# H2OWorld - Building Machine Learning Applications with Sparkling Water - forked from:
[h2O source](https://github.com/h2oai/h2o-tutorials/blob/master/tutorials/sparkling-water/README.md)

1. Definindo as variáveis de ambiente
```
export SPARK_HOME="/opt/spark/spark-1.5.1-bin-hadoop2.6"
export MASTER="local-cluster[3,2,4096]"
/opt/spark/sparkling-water-1.5.6/bin/sparkling-shell  --conf "spark.executor.memory=2g"
```
2. Carregando os dados
```
// Input data
val DATAFILE="./data/smsData.txt"

```
3. Importando as bibliotecas:
```
// Common imports from H2O and Sparks
import _root_.hex.deeplearning.{DeepLearningModel, DeepLearning}
import _root_.hex.deeplearning.DeepLearningParameters
import org.apache.spark.examples.h2o.DemoUtils._
import org.apache.spark.h2o._
import org.apache.spark.mllib
import org.apache.spark.mllib.feature.{IDFModel, IDF, HashingTF}
import org.apache.spark.rdd.RDD
import water.Key
```
4. Definindo uma mensagem de treino
```
// Representation of a training message
case class SMS(target: String, fv: mllib.linalg.Vector)
```
5. Definindo o data loader e parser:
```
def load(dataFile: String): RDD[Array[String]] = {
  // Load file into memory, split on TABs and filter all empty lines
  sc.textFile(dataFile).map(l => l.split("\t")).filter(r => !r(0).isEmpty)
}
```
6. Definindo o tokenizer:
```
// Tokenizer
// For each sentence in input RDD it provides array of string representing individual interesting words in the sentence
def tokenize(dataRDD: RDD[String]): RDD[Seq[String]] = {
  // Ignore all useless words
  val ignoredWords = Seq("the", "a", "", "in", "on", "at", "as", "not", "for")
  // Ignore all useless characters
  val ignoredChars = Seq(',', ':', ';', '/', '<', '>', '"', '.', '(', ')', '?', '-', '\'','!','0', '1')

  // Invoke RDD API and transform input data
  val textsRDD = dataRDD.map( r => {
    // Get rid of all useless characters
    var smsText = r.toLowerCase
    for( c <- ignoredChars) {
      smsText = smsText.replace(c, ' ')
    }
    // Remove empty and uninteresting words
    val words = smsText.split(" ").filter(w => !ignoredWords.contains(w) && w.length>2).distinct

    words.toSeq
  })
  textsRDD
}
```
7. Tf-IDF modeler:
```
def buildIDFModel(tokensRDD: RDD[Seq[String]],
                  minDocFreq:Int = 4,
                  hashSpaceSize:Int = 1 << 10): (HashingTF, IDFModel, RDD[mllib.linalg.Vector]) = {
  // Hash strings into the given space
  val hashingTF = new HashingTF(hashSpaceSize)
  val tf = hashingTF.transform(tokensRDD)
  
  // Build term frequency-inverse document frequency model
  val idfModel = new IDF(minDocFreq = minDocFreq).fit(tf)
  val expandedTextRDD = idfModel.transform(tf)
  (hashingTF, idfModel, expandedTextRDD)
}
```
8. Configurando o modelo de deep learning d a H2O:
```
def buildDLModel(trainHF: Frame, validHF: Frame,
               epochs: Int = 10, l1: Double = 0.001, l2: Double = 0.0,
               hidden: Array[Int] = Array[Int](200, 200))
              (implicit h2oContext: H2OContext): DeepLearningModel = {
  import h2oContext._
  import _root_.hex.deeplearning.DeepLearning
  import _root_.hex.deeplearning.DeepLearningParameters
  // Create algorithm parameres
  val dlParams = new DeepLearningParameters()
  // Name for target model
  dlParams._model_id = Key.make("dlModel.hex")
  // Training dataset
  dlParams._train = trainHF
  // Validation dataset
  dlParams._valid = validHF
  // Column used as target for training
  dlParams._response_column = 'target
  // Number of passes over data
  dlParams._epochs = epochs
  // L1 penalty
  dlParams._l1 = l1
  // Number internal hidden layers
  dlParams._hidden = hidden

  // Create a DeepLearning job
  val dl = new DeepLearning(dlParams)
  // And launch it
  val dlModel = dl.trainModel.get

  // Force computation of model metrics on both datasets
  dlModel.score(trainHF).delete()
  dlModel.score(validHF).delete()

  // And return resulting model
  dlModel
}
```
9. Inicializando o contexto de h2OContext e iniciando os serviços da H2O em cima da arquitetura do Spark: 
```
// Create SQL support
 import org.apache.spark.sql._
 implicit val sqlContext = SQLContext.getOrCreate(sc)
 import sqlContext.implicits._

 // Start H2O services
 import org.apache.spark.h2o._
 val h2oContext = new H2OContext(sc).start()
```
10. Abrindo o console da H20 e verificar que o H2O está rodando:
```
h2oContext.openFlow
//O comando ```getCloud``` permite ver o status da h2O cloud. 
```
11. Construindo o workflow final utilizando as funções previamente definidas:
```
// Data load
val dataRDD = load(DATAFILE)
// Extract response column from dataset
val hamSpamRDD = dataRDD.map( r => r(0))
// Extract message from dataset
val messageRDD = dataRDD.map( r => r(1))
// Tokenize message content
val tokensRDD = tokenize(messageRDD)

// Build IDF model on tokenized messages
// It returns
//   - hashingTF: hashing function to hash a word to a vector space
//   - idfModel: a model to transform hashed sentence to a feature vector
//   - tfidf: transformed input messages
var (hashingTF, idfModel, tfidfRDD) = buildIDFModel(tokensRDD)

// Merge response with extracted vectors
val resultDF = hamSpamRDD.zip(tfidfRDD).map(v => SMS(v._1, v._2)).toDF

// Publish Spark DataFrame as H2OFrame  
val tableHF = h2oContext.asH2OFrame(resultDF, "messages_table")

// Transform target column into categorical!
tableHF.replace(tableHF.find("target"), tableHF.vec("target").toCategoricalVec()).remove()
tableHF.update(null)

// Split table into training and validation parts
val keys = Array[String]("train.hex", "valid.hex")
val ratios = Array[Double](0.8)
val frs = split(tableHF, keys, ratios)
val (trainHF, validHF) = (frs(0), frs(1))
tableHF.delete()

// Build final DeepLearning model
val dlModel = buildDLModel(trainHF, validHF)(h2oContext)
```
12. 
