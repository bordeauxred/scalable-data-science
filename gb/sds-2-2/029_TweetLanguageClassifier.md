[SDS-2.2, Scalable Data Science](https://lamastex.github.io/scalable-data-science/sds/2/2/)
===========================================================================================

Twitter Streaming Language Classifier
=====================================

This is a databricksification of <https://databricks.gitbooks.io/databricks-spark-reference-applications/content/twitter_classifier/index.html> by Amendra Shreshta.

Note that you need to change the fields in background notebooks like `025_a_extendedTwitterUtils2run` as explained in the corresponding videos by Amendra.

> import twitter4j.\_ import twitter4j.auth.Authorization import twitter4j.conf.ConfigurationBuilder import twitter4j.auth.OAuthAuthorization import org.apache.spark.streaming.\_ import org.apache.spark.streaming.dstream.\_ import org.apache.spark.storage.StorageLevel import org.apache.spark.streaming.receiver.Receiver

    import org.apache.spark._
    import org.apache.spark.storage._
    import org.apache.spark.streaming._

    import scala.math.Ordering

    import twitter4j.auth.OAuthAuthorization
    import twitter4j.conf.ConfigurationBuilder

> import org.apache.spark.\_ import org.apache.spark.storage.\_ import org.apache.spark.streaming.\_ import scala.math.Ordering import twitter4j.auth.OAuthAuthorization import twitter4j.conf.ConfigurationBuilder

> defined class ExtendedTwitterReceiver

> defined class ExtendedTwitterInputDStream

> import twitter4j.Status import twitter4j.auth.Authorization import org.apache.spark.storage.StorageLevel import org.apache.spark.streaming.StreamingContext import org.apache.spark.streaming.dstream.{ReceiverInputDStream, DStream} defined object ExtendedTwitterUtils

> done running the extendedTwitterUtils2run notebook - ready to stream from twitter

    import twitter4j.auth.OAuthAuthorization
    import twitter4j.conf.ConfigurationBuilder

    def MyconsumerKey       = "fB9Ww8Z4TIauPWKNPL6IN7xqd"
    def MyconsumerSecret    = "HQqiIs3Yx3Mnv5gZTwQ6H2DsTlae4UNry5uNgylsonpFr46qXy"
    def Mytoken             = "28513570-BfZrGoswVp1bz11mhwbVIGoJwjWCWgGoZGQXAqCO8"
    def MytokenSecret       = "7fvag0GcXRlv42yBaVDMAmL1bmPyMZzNrqioMY7UwGbxr"

    System.setProperty("twitter4j.oauth.consumerKey", MyconsumerKey)
    System.setProperty("twitter4j.oauth.consumerSecret", MyconsumerSecret)
    System.setProperty("twitter4j.oauth.accessToken", Mytoken)
    System.setProperty("twitter4j.oauth.accessTokenSecret", MytokenSecret)

> import twitter4j.auth.OAuthAuthorization import twitter4j.conf.ConfigurationBuilder MyconsumerKey: String MyconsumerSecret: String Mytoken: String MytokenSecret: String res1: String = null

    // Downloading tweets and building model for clustering

    // ## Let's create a directory in dbfs for storing tweets in the cluster's distributed file system.
    val outputDirectoryRoot = "/datasets/tweetsStreamTmp" // output directory

> outputDirectoryRoot: String = /datasets/tweetsStreamTmp

    // to remove a pre-existing directory and start from scratch uncomment next line and evaluate this cell
    dbutils.fs.rm(outputDirectoryRoot, true) 

> res2: Boolean = false

    // ## Capture tweets in every sliding window of slideInterval many milliseconds.
    val slideInterval = new Duration(1 * 1000) // 1 * 1000 = 1000 milli-seconds = 1 sec

> slideInterval: org.apache.spark.streaming.Duration = 1000 ms

    // Our goal is to take each RDD in the twitter DStream and write it as a json file in our dbfs.
    // Create a Spark Streaming Context.
    val ssc = new StreamingContext(sc, slideInterval)

> ssc: org.apache.spark.streaming.StreamingContext = org.apache.spark.streaming.StreamingContext@3e2a07fd

    // Create a Twitter Stream for the input source. 
    val auth = Some(new OAuthAuthorization(new ConfigurationBuilder().build()))
    val twitterStream = ExtendedTwitterUtils.createStream(ssc, auth)

> auth: Some\[twitter4j.auth.OAuthAuthorization\] = Some(OAuthAuthorization{consumerKey='fB9Ww8Z4TIauPWKNPL6IN7xqd', consumerSecret='\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*', oauthToken=AccessToken{screenName='null', userId=28513570}}) twitterStream: org.apache.spark.streaming.dstream.ReceiverInputDStream\[twitter4j.Status\] = ExtendedTwitterInputDStream@2acbe39c

    // Let's import google's json library next.
    import com.google.gson.Gson 
    //Let's map the tweets into json formatted string (one tweet per line).
    val twitterStreamJson = twitterStream.map(
                                                x => { val gson = new Gson();
                                                     val xJson = gson.toJson(x)
                                                     xJson
                                                     }
                                              ) 

> import com.google.gson.Gson twitterStreamJson: org.apache.spark.streaming.dstream.DStream\[String\] = org.apache.spark.streaming.dstream.MappedDStream@616fc13d

    val partitionsEachInterval = 1 

    val batchInterval = 1 // in minutes
    val timeoutJobLength =  batchInterval * 5

    var newContextCreated = false
    var numTweetsCollected = 0L // track number of tweets collected

    twitterStreamJson.foreachRDD((rdd, time) => { // for each filtered RDD in the DStream
          val count = rdd.count()
          if (count > 0) {
            val outputRDD = rdd.repartition(partitionsEachInterval) // repartition as desired
            // to write to parquet directly in append mode in one directory per 'time'------------       
            val outputDF = outputRDD.toDF("tweetAsJsonString")
            // get some time fields from current `.Date()`
            val year = (new java.text.SimpleDateFormat("yyyy")).format(new java.util.Date())
            val month = (new java.text.SimpleDateFormat("MM")).format(new java.util.Date())
            val day = (new java.text.SimpleDateFormat("dd")).format(new java.util.Date())
            val hour = (new java.text.SimpleDateFormat("HH")).format(new java.util.Date())
            // write to a file with a clear time-based hierarchical directory structure for example
            outputDF.write.mode(SaveMode.Append)
                    .parquet(outputDirectoryRoot+ "/"+ year + "/" + month + "/" + day + "/" + hour + "/" + time.milliseconds) 
            // end of writing as parquet file-------------------------------------
            numTweetsCollected += count // update with the latest count
          }
      })

> partitionsEachInterval: Int = 1 batchInterval: Int = 1 timeoutJobLength: Int = 5 newContextCreated: Boolean = false numTweetsCollected: Long = 0

    // ## Let's start the spark streaming context we have created next.
    ssc.start()

    // total tweets downloaded
    numTweetsCollected

> res13: Long = 1836

    // ## Go to SparkUI and see if a streaming job is already running. If so you need to terminate it before starting a new streaming job. Only one streaming job can be run on the DB CE.
    // #  let's stop the streaming job next.
    ssc.stop(stopSparkContext = false) 
    StreamingContext.getActive.foreach { _.stop(stopSparkContext = false) } 

    // #Let's examine what was saved in dbfs
    display(dbutils.fs.ls(outputDirectoryRoot))

| dbfs:/datasets/tweetsStreamTmp/2017/ | 2017/ | 0.0 |
|--------------------------------------|-------|-----|

    // Replace the date with current date
    val date = "/2017/11/*"
    val rawDF = fromParquetFile2DF(outputDirectoryRoot + date +"/*/*") //.cache()
    val TTTsDF = tweetsDF2TTTDF(tweetsJsonStringDF2TweetsDF(rawDF)).cache()

> date: String = /2017/11/\* rawDF: org.apache.spark.sql.DataFrame = \[tweetAsJsonString: string\] TTTsDF: org.apache.spark.sql.Dataset\[org.apache.spark.sql.Row\] = \[CurrentTweetDate: timestamp, CurrentTwID: bigint ... 33 more fields\]

    // Creating SQL table 
    TTTsDF.createOrReplaceTempView("tbl_tweet")

    sqlContext.sql("SELECT lang, CPostUserName, CurrentTweet FROM tbl_tweet LIMIT 10").collect.foreach(println)

> \[ja,��NAT�UKI��,RT @she\_is\_lie: https://t.co/aGTKqpjHva\] \[en,TesiaD-1 WSD�,Not that it matters but 38 minutes until I turn 18\] \[ja,����,@clubj\_ $��]�j�`Wm�\] \[en,Pratik Raj IN,@ZeeNewsHindi Is apna muh bhi kala karwana chahiye Tha agar asal ki virodhi Hai to @MamataOfficial\] \[ja,jJ,�FD�\] \[it,.,RT @baciamicoglione: m i p i a c e l a f i g a\] \[en,Mehboob,RT @raheelrana: E�'� 5'-( *3� H� 41�A *H'�� H��D H� 41�A ./' /' H'37� ,� "~ '~F� "~ �H E* (/DH E�1 '~F� H��D �H (/D DH ,3 71-& \] \[en,M�����,RT @jlist: When things are going really bad at work. https://t.co/0cqPLeKcPX\] \[ja,jyi�,AFA in �~) Zch�Df_�fnw�󦧤 ����k3��`BK���Lb~�jD �U�n1pt1pt����WfO`UD https://t.co/GcrQYqJ1MP \#H�j https://t.co/XsCIFqxWbQ\] \[ja,��(8rzC),RT @Kono0425\_ry: �n�JdQfUD �k�dQ�( ,�)n��L�f�QkwMf~Y �n�@o�h`K�h��[Zk�zkec_����dQ_�Y�������W~Y �M��`YhUAU~gY& \]

    // Checking the language of tweets
    sqlContext.sql(
        "SELECT lang, COUNT(*) as cnt FROM tbl_tweet " +
        "GROUP BY lang ORDER BY cnt DESC limit 1000")
        .collect.foreach(println)

> \[en,626\] \[ja,513\] \[ko,142\] \[ar,107\] \[es,94\] \[pt,72\] \[th,67\] \[fr,49\] \[tr,38\] \[ru,31\] \[it,18\] \[ca,17\] \[id,16\] \[en-gb,13\] \[de,13\] \[zh-cn,10\] \[nl,8\] \[zh-CN,3\] \[fi,3\] \[sr,2\] \[hu,2\] \[el,2\] \[zh-TW,2\] \[en-GB,2\] \[pl,2\] \[vi,2\] \[zh-tw,1\] \[ro,1\] \[hr,1\] \[uk,1\] \[bg,1\] \[en-AU,1\] \[zh-Hant,1\] \[hi,1\] \[da,1\]

    // extracting just tweets from the table and converting it to String
    val texts = sqlContext
          .sql("SELECT CurrentTweet from tbl_tweet")
          .map(_.toString)

> texts: org.apache.spark.sql.Dataset\[String\] = \[value: string\]

    import org.apache.spark.mllib.clustering.KMeans
    import org.apache.spark.mllib.linalg.{Vector, Vectors}

> import org.apache.spark.mllib.clustering.KMeans import org.apache.spark.mllib.linalg.{Vector, Vectors}

    /*Create feature vectors by turning each tweet into bigrams of characters (an n-gram model)
    and then hashing those to a length-1000 feature vector that we can pass to MLlib.*/

    def featurize(s: String): Vector = {
      val n = 1000
      val result = new Array[Double](n)
      val bigrams = s.sliding(2).toArray
      for (h <- bigrams.map(_.hashCode % n)) {
        result(h) += 1.0 / bigrams.length
      }
      Vectors.sparse(n, result.zipWithIndex.filter(_._1 != 0).map(_.swap))
    }

> featurize: (s: String)org.apache.spark.mllib.linalg.Vector

    //Cache the vectors RDD since it will be used for all the KMeans iterations.
    val vectors = texts.rdd
          .map(featurize)
          .cache()

> vectors: org.apache.spark.rdd.RDD\[org.apache.spark.mllib.linalg.Vector\] = MapPartitionsRDD\[787\] at map at command-2771931608832193:3

    // cache is lazy so count will force the data to store in memory
    vectors.count()

> res21: Long = 1863

    vectors.first()

> res22: org.apache.spark.mllib.linalg.Vector = (1000,\[50,53,56,78,96,99,100,180,189,226,285,325,340,350,356,358,370,438,453,488,504,525,554,573,578,587,615,623,626,636,642,660,669,679,708,712,755,830,845,903\],\[0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025,0.025\])

    // Training model with 10 cluster and 10 iteration
    val model = KMeans.train(vectors, k=10, maxIterations = 10)

> model: org.apache.spark.mllib.clustering.KMeansModel = org.apache.spark.mllib.clustering.KMeansModel@616925c8

    // Sample 100 of the original set
    val some_tweets = texts.take(100)

> some\_tweets: Array\[String\] = Array(\[RT @she\_is\_lie: https://t.co/aGTKqpjHva\], \[Not that it matters but 38 minutes until I turn 18\], \[@clubj\_ $��]�j�`Wm�\], \[@ZeeNewsHindi Is apna muh bhi kala karwana chahiye Tha agar asal ki virodhi Hai to @MamataOfficial\], \[�FD�\], \[RT @baciamicoglione: m i p i a c e l a f i g a\], \[RT @raheelrana: E�'� 5'-( *3� H� 41�A *H'�� H��D H� 41�A ./' /' H'37� ,� "~ '~F� "~ �H E* (/DH E�1 '~F� H��D �H (/D DH ,3 71-& \], \[RT @jlist: When things are going really bad at work. https://t.co/0cqPLeKcPX\], \[AFA in �~) Zch�Df_�fnw�󦧤 ����k3��`BK���Lb~�jD �U�n1pt1pt����WfO`UD https://t.co/GcrQYqJ1MP \#H�j https://t.co/XsCIFqxWbQ\], \[RT @Kono0425\_ry: �n�JdQfUD �k�dQ�( ,�)n��L�f�QkwMf~Y �n�@o�h`K�h��[Zk�zkec_����dQ_�Y�������W~Y �M��`YhUAU~gY& \], \[���Ċn�᯸gY ��jPsP ^}np`Q'VoB�^ | KJ�X \#pixiv��ï https://t.co/UJOQWDqp58\], \[RT @whatgirIsIove: no offence to me but wtf am i doing\], \[RT @yuyu\_d: \#! gN�u�BR��_���� https://t.co/UFiaVVfHcj\], \[(ADE' ,'! #E1F' ,9DF' 9'DJG' 3'ADG' H#E71F' 9DJG' -,'1) EF 3,JD EF6H/) \[GH/:82\] https://t.co/HTLfiMcgb3\], \[1(J #9H0 (C EF 'DC3D H3H! 'DC(1 https://t.co/jCbc2qxOlI\], \[RT @bellyinsmile: %H-!I-- \#
9C https://t.co/XmIecEtLLh\], \[RT @chortletown: �Pledge 4my life 
U��U�n\#UW�W
K in Nagoya-shi, � https://t.co/JAvvHX85nt\], \[RT @KylesHotTakes: And Doc Halladay grounds out to end the inning\], \[*BHDJ E' *3*-JF 9DI H,G, E5(:) #8'A1 1,HD, 4FH J9FJ ':1'! ! '0(-G'\], \[bla bla bla https://t.co/1VmXZk9rRH\], \[@Aghaye\_Biikhat '1G /�/E\], \[@jalalaeddine @S\_ALKarashi 3(-'F 'DDG */'A9 9F 'D5HAJ) H-2( 'D(9+ 'D%4*1'CJ... H*1J/ EF' #F FB(D CD'EC\], \[@Kiekkokirja Livetuloksista. En oo 100% varma onko tuo totta kun siell� on joskus v��r�� tietoa esim. kokoonpanoissa\], \[���go ۳L�fjD(T^T) ��`h����U�L �7Of�f~YL(c��\`c)\], \[@carol19761112 J<Wf�DfM~W_:�DgY�m(\], \[@kero\_hugu �� Bj_koy%����UWBR~Yd y%����gKFC�n���}W�gmW 傭�����k Wf���n��netd
8C+
J�\], \[RT @\_Tazmxni: Mon pote il m'accompagne au magasin et pour l'remercier j'lui dis prend un bail, ce fou il avait l'intention d'acheter une co&\], \[RT @taekookmoments: can't believe taekook checked what letter each other got to see which team they're in wHAT @@ https://t.co/dsNi9QLzJS\], \[RT @famichikisenpai: \#i�u[S�D 	jͿDcqDcfM_&�U��O��cfO`UD � թ��&RTg3
-7H!I3-1%!!21IAH@G 7H!A%I'04F#4@'1+I2& \], \[Bluetooth����jOW_SDdA��ïgK�\_(-�-\`\_))\_\], \[Sendo fofas na Disneyland!! ��

    // iterate through the 100 samples and show which cluster they are in
    for (i <- 0 until 10) {
      println(s"\nCLUSTER $i:")
      some_tweets.foreach { t =>
        if (model.predict(featurize(t)) == i) {
          println(t)
        }
      }
    }

> CLUSTER 0: \[RT @she\_is\_lie: https://t.co/aGTKqpjHva\] \[RT @jlist: When things are going really bad at work. https://t.co/0cqPLeKcPX\] \[AFA in �~) Zch�Df_�fnw�󦧤 ����k3��`BK���Lb~�jD �U�n1pt1pt����WfO`UD https://t.co/GcrQYqJ1MP \#H�j https://t.co/XsCIFqxWbQ\] \[���Ċn�᯸gY ��jPsP ^}np`Q'VoB�^ | KJ�X \#pixiv��ï https://t.co/UJOQWDqp58\] \[RT @yuyu\_d: \#! gN�u�BR��_���� https://t.co/UFiaVVfHcj\] \[(ADE' ,'! #E1F' ,9DF' 9'DJG' 3'ADG' H#E71F' 9DJG' -,'1) EF 3,JD EF6H/) \[GH/:82\] https://t.co/HTLfiMcgb3\] \[1(J #9H0 (C EF 'DC3D H3H! 'DC(1 https://t.co/jCbc2qxOlI\] \[RT @bellyinsmile: %H-!I-- \#
9C https://t.co/XmIecEtLLh\] \[RT @chortletown: �Pledge 4my life 
K in Nagoya-shi, � https://t.co/JAvvHX85nt\] \[bla bla bla https://t.co/1VmXZk9rRH\] \[@kero\_hugu �� Bj_koy%����UWBR~Yd y%����gKFC�n���}W�gmW 傭�����k Wf���n��netd
8C+
U��U�n\#UW�W
J�\] \[RT @famichikisenpai: \#i�u[S�D 	jͿDcqDcfM_&�U��O��cfO`UD � թ��&RTg3
-7H!I3-1%!!21IAH@G 7H!A%I'04F#4@'1+I2& \] \[Bluetooth����jOW_SDdA��ïgK�\_(-�-\`\_))\_\] \[RT @CrushOn2226: 20171107 \#MONSTA\_X \#���Ѥ \#Dt� \#IM \#0 \#KIHYUN \#DRAMARAMA \#�|�|� @OfficialMonstaX SHOW CON DRAMARAMA 0 focus ful& \] \[JKn�YV���L^L�nLK?�.

    // to remove a pre-existing model and start from scratch
    dbutils.fs.rm("/datasets/model", true) 

> res24: Boolean = false

    // save the model
    sc.makeRDD(model.clusterCenters).saveAsObjectFile("/datasets/model")

    import org.apache.spark.mllib.clustering.KMeans
    import org.apache.spark.mllib.linalg.{Vector, Vectors}
    import org.apache.spark.mllib.clustering.KMeansModel

> import org.apache.spark.mllib.clustering.KMeans import org.apache.spark.mllib.linalg.{Vector, Vectors} import org.apache.spark.mllib.clustering.KMeansModel

    // Checking if the model works
    val clusterNumber = 5

    val modelFile = "/datasets/model"

    val model: KMeansModel = new KMeansModel(sc.objectFile[Vector](modelFile).collect)
    model.predict(featurize("H'-/ 5'-(I DH -/ J91A 'CHF* H2J1 'D*9DJE ")) == clusterNumber

> clusterNumber: Int = 5 modelFile: String = /datasets/model model: org.apache.spark.mllib.clustering.KMeansModel = org.apache.spark.mllib.clustering.KMeansModel@4b53f956 res26: Boolean = false

> USAGE: val df = tweetsDF2TTTDF(tweetsJsonStringDF2TweetsDF(fromParquetFile2DF("parquetFileName"))) val df = tweetsDF2TTTDF(tweetsIDLong\_JsonStringPairDF2TweetsDF(fromParquetFile2DF("parquetFileName"))) import org.apache.spark.sql.types.{StructType, StructField, StringType} import org.apache.spark.sql.functions.\_ import org.apache.spark.sql.types.\_ import org.apache.spark.sql.ColumnName import org.apache.spark.sql.DataFrame fromParquetFile2DF: (InputDFAsParquetFilePatternString: String)org.apache.spark.sql.DataFrame tweetsJsonStringDF2TweetsDF: (tweetsAsJsonStringInputDF: org.apache.spark.sql.DataFrame)org.apache.spark.sql.DataFrame tweetsIDLong\_JsonStringPairDF2TweetsDF: (tweetsAsIDLong\_JsonStringInputDF: org.apache.spark.sql.DataFrame)org.apache.spark.sql.DataFrame tweetsDF2TTTDF: (tweetsInputDF: org.apache.spark.sql.DataFrame)org.apache.spark.sql.DataFrame tweetsDF2TTTDFWithURLsAndHastags: (tweetsInputDF: org.apache.spark.sql.DataFrame)org.apache.spark.sql.DataFrame

    // Loading model and printing tweets that matched the desired cluster

    var newContextCreated = false
    var num = 0

    // Create a Spark Streaming Context.
    @transient val ssc = new StreamingContext(sc, slideInterval)
    // Create a Twitter Stream for the input source. 
    @transient val auth = Some(new OAuthAuthorization(new ConfigurationBuilder().build()))
    @transient val twitterStream = ExtendedTwitterUtils.createStream(ssc, auth)

    //Replace the cluster number as you desire between 0 to 9
    val clusterNumber = 6

    //model location
    val modelFile = "/datasets/model"

    // Get tweets from twitter
    val Tweet = twitterStream.map(_.getText)
    //Tweet.print()

    println("Initalizaing the the KMeans model...")
    val model: KMeansModel = new KMeansModel(sc.objectFile[Vector](modelFile).collect)

    //printing tweets that match our choosen cluster
    Tweet.foreachRDD(rdd => {  
    rdd.collect().foreach(i =>
        {
           val record = i
           if (model.predict(featurize(record)) == clusterNumber) {
           println(record)
        }
        })
    })

    // Start the streaming computation
    println("Initialization complete.")
    ssc.start()
    ssc.awaitTermination()

> Initalizaing the the KMeans model... Initialization complete. @tiasakuraa\_ @rnyx\_x Yeahhhh geng. No more stress anymore. Let's happy2 till the end of this year p RT @FlowerSree: Go for it now...the future is promised to no one. ~Wayne Dyer~ https://t.co/t6XqmROnxg Paket Nasi Putih M Thanks ya Mba udh repeat orderannya	 utk Thalilan Bapak& https://t.co/3T00NxVAxi RT @davidschneider: Watch out, Brexiters! We Remoaners can force you to go to clandestine meetings then take control of your mouth and& @nitinkumarjdhav @nitin\_gadkari @narendramodi @BJP4India @PMOIndia @PIB\_India @MIB\_India Bhai jio sim lene me hi 4 jaan chali gayi, Uska kya? RT @xiskenderx: Nas1ls1n1z? IYI'yim! 0yiyim diyenler dikkat! � � Bir kamyonun �arpmas1yla yaralanm1_ olan �ift�i Mehmet amca... https://t& RT @BoobsTitsOppai: @hondarb4p906i @OjifTakaoka @onitarou5678 @lcaligula @singlelife43 @EronaviInfo @JapaneseAVBabes @NaughtyTokyo& RT @sigCapricornio2: \#Capric�rnio "�s vezes a vida perde a gra�a." RT @buenos\_nialles: jak harry nic nie wspomni o polsce majc taki widok to przechodze do fandomu krzysztofa krawczyka& RT @RakanMasjidUSM: Wahai mata, dah brp lama kau tidak menangis kerana memikirkan dosa, syurga, neraka & rahmat Dia? Hati, kau apa khabar?& RT @kiaralawhon: Toxic relationships hinder growth. & as women, we want to take care of our men and sometimes forget that we deserve& NTEC COMPACT MULTI PLAYER ADAPTOR - MULTI TAP - Brand new! For use with the PS2 https://t.co/g61m1o7CpF \#ebay \#gamer \#gaming \#games RT @CaratLandPH: Seungcheol, Jeonghan & Jisoo riding a tricycle!�� A concept PH-Carats live for.� Kyah kyah penge pambarya: https://t.c& @AndyMaherDFA @Telstra They are a genuine joke. Had a woman call me back twice because I couldnt log into the Telstra app. She then advised me to seek help via the website live chat option. In the end I just thought fuck it, I dont need the app. \#easier @iHyungseob\_ Sa ae sia borokokok :( ,kangen ugha mwahh 2 math exams for the first week yEs polynomials !!! RT @Assimalhakeem: You must at least make sure it is not pork you are eating. Avoiding it is best. https://t.co/k6VCgqWomd RT @HQGatica: a @tvn le inyecten 47 millones de d�lares y ni siquiera levantan cr�tica a la directiva por su p�sima administraci�n y sueldo& RT @khalheda: g envie daller au bord dun lac avc un beau gar�on en d�capotable et on regarde la lune et apr�s jle noie et je pars avec la& RT @SESAR\_JU: J. Pie @ASDEurope how do we increase the appetite for European project for aviation? we need to work together to re& im gonna go ahead and say it..... kali in ST2 was done so dirty and should have had more screen time Theres really no good reason not to do this https://t.co/uGO2Af5hTu 5`c_P��Dj RT @zhukl: Haven't seen this kind of Islam in awhile. Welcome back. I missed you. https://t.co/7YuZ3iYvOG @awlazel @Schroedinger99 @PsychFarmer @nick\_gutteridge @valko665 If you're saying the EU would collectively boot out all Brits because we reject the ECJ, it says more about the EU than it does about the UK. RT @maaripatriota: sinto sdd de umas fases da minha vida q eu rezava p passar logo e q hj em dia eu daria tudo p ter dnv @Zoutdropje ik hou altijd 2 aan... 2 repen per persoon @vladsavov Do you mind emailing me the first and the third in high-res? I would like to enjoy the full quality on my wallpaper. Thanks! @missm\_girl Se ainda fosse numas maminhas. Pfffff @orangetwinspur Uu nga Ate C gusto ko na mag g! Hip Hop Is Not Just "Rap".... It's A Whole Culture. @Abyss\_59\_cide Ka~W_ X el asma chota esta no dormi en toda la noche,pobres los que me tienen q ver la cara de culo @Im\_Myystic @unlikelyheroI @thandamentalist dekho kya hota hai. This site you can be purchased at cheap topic of PC games. Discount code, there are deals such as the sale information. https://t.co/UEpa4OHTpe RT @ShaunKing: 38 year old Melvin Carter just became the first Black Mayor of St. Paul, Minnesota. https://t.co/rreMZAfBNa @Cultura harry p�teur ? Yeah, I suck at RL but let's make a safer gaming community.. let's be a part of the solution not a part of the problem. & RT @BUNDjugend: \#pre2020 \#climateactionsnow in front of entry to \#BulaZone at \#COP23 with @DClimateJustice and some oft us https://t.co/yTj& RT @acgrayling: (2) But! - once the lunacy of Brexit is stopped, we can clean the Augean stables of our political order, & get our democrac& Serious la lupa yang aku ni perempuan - The Hornets are sadly not playing today so are off to do some bowling!  But Good Luck to all our @hudunisport teams today @HURLFC @UHRUFC @HULF1 @\_HUFC @HUNetball @HuddsHawks @UoHHockey � we'll see you all out later tonight { \#huddersfield Vroeger herkende je de Sint aan zijn schoenen en zwarte Piet aan zijn stem maar dat geheel terzijde. Er zijn daarentegen Pieten waarvan je denkt Who the Fuck is dat nou weer. https://t.co/nT7NmOso2T RT @iamwilliewill: Need to vent? Get in the shower Need to practice a future argument? Get in the shower Need some alone time? Get& @deanxshah lain baca lain keluar hanat If @HEARTDEFENSOR and @ATelagaarta aren't goals, then idk what is. https://t.co/0NSfONADna RT @La\_Directa: @CDRCatOficial \#VagaGeneral8N | L'avinguda Diagonal \#BCN tamb� est� tallada per un piquet d'unes 200 persones& RT @7horseracing: WATCH: Take a @ at the \#MelbourneCup from a jockey's perspective, on-board w/@dwaynedunn8 , as he guides home 8th p& RT @javierjuantur: Gallard�n, es \#CosaNostra. Ignacio Gonz�lez acusado de tapar los delitos del Canal en la etapa de Gallard�n.& I won 3 achievements in Wolfenstein II: The New Colossus for 26 \#TrueAchievement pts https://t.co/K2re2ox59h Ugo Ehiogu: Football Black List to name award after ex-England defender \#Birmingham https://t.co/67SoR1RoK3 @Channel4News @carol580532 @EmilyThornberry @Anna\_Soubry @BorisJohnson The time has come for @Boris Johnson to resign, he has do e enough damage no more allowances made 4 him @HiRo\_borderline MfD~[�� RT @caedeminthecity: @StarMusicPH @YouTube Wahhhhhhh! Yas! I guess this is the go ahead to use: @mor1019 \#MORPinoyBiga10 Alam Na This b& RT @cadenguezinha: s� queria dormir pra sempre tipo morrer sabe Ran�o tem sido meu alimento noite e dia. RT @sayitoh: Fumen, chupen, cojan, se van a morir igual porque los Etchevehere del pa�s ya les envenenaron el mayor r�o argentin& @Balancement @The\_music\_gala Probabilmente si , anche se alcuni storici nutrono dubbi. Ma non importa....lo ricordiamo per i suoi Capolavori RT @SEGOB\_mx: �Qu� es la ceniza volc�nica? y �C�mo puede afectar a tu salud? \#Inf�rmate https://t.co/HxYVUMDqWb @PcSegob& Screaming told my grandma Im working in Town tonight and I quote TOON TOON TOON was the message that followed ahahaaa @\_goodstranger nao da pra desperdi�ar dinheiro beijando RT @bellaspaisitas: �Impresionantes! Descubren las caracter�sticas de las mujeres m�s inteligentes!11https://t.co/0zgpP6uKKS Tan1mad11m biri gelip hal hat1r sorunca tan1yomu_um gibi konu_may1 devam ettiririm, bu da benim serbest serseri stilim D RT @Dustman\_Dj: when Your Dad's WhatsApp profile pic are kids you don't even know https://t.co/JfcPo017AJ wait - forever na ba to ? may convo na kami ni crush H RT @Monica\_Wilcox: That's the thing about books. They let you travel without moving your feet. ~Jhumpa Lahiri, The Namesake \#art-The& @Kylapatricia4 @lnteGritty @succ\_ulent\_1 @Patrickesque @ZippersNYC @HumorlessKev @pang5 What are the policies that Our Revolution supports? RT @cutenessAvie: SHAME SHAME SHAME SHAME SHAME SHAME SHAME SHAME SHAME SHAME SHAME SHAME SHAME SHAME SHAME SHAME SHAME SHAME SHAME S& @cupnacional Huelga es actos vandalicos y obligar al resto a no circular o coger transportes? A ver si empieza la GC a poner orden y se os acaba la tonteria, q ya no queda paciencia y algun vecino os lo va a explicar de manera clara y rapida... RT @JackPosobiec: Fmr Gillespie staffer to me: I walked out before the primary when I saw the campaign was run by NeverTrump losers @AuthorSJBolton @JenLovesReading @ElaineEgan\_ @sarahbenton @SamEades Amazing! CHP'nin Meclis Ba_kan aday1 farketmez, etkisi s1f1r olacak ��nk�. RT @rasadangubic: Apsolutno se u ovaj dijalog moraju uklju
 
 OCEAN MAN 
 
 The latest The Susan Lewis Daily! https://t.co/KeLViquq0g Thanks to @Wildmoonsister @ScientologyVM @Tamaraw68415067 \#amwriting \#writerslife @Soratania ...tutti scegliamo in base all'umore... mai promesse quando si � felici.. e mai decisioni quando si � incazzati... @JustSafwan Diam le wahai anak engineering 'De Bill Cosby a Kevin Spacey: los esc�ndalos sexuales de Hollywood a trav�s del tiempo ' https://t.co/3wGStOqzVv v�a @RevistaSemana RT @andrewrodge6: college is the razor scooter & I am the ankle Da hast du \#280Zeichen zur Verf�gung und denkst, das ist was besonderes, da hat's dann jeder. Nix mit Fame. Fast 7 Jahre Twitter und dann das. F�hlt sich verboten und falsch an. So wie hinterm Vorhang. So, das war mein erster 280er Tweet. Jetzt erstmal weiter fr�hst�cken. This Man?s Enormous Blackheads Were Popped Out And The Results Are Disturbing. https://t.co/BP1BD8lxrr RT @Larissacostas: A ver si as� se entiende, porque parece que no les queda claro  https://t.co/JBSAXKly7T @wepl4ydumb Ela dormiu de boca no Jubileu ??? RT @iaccrazyzhang: @harrypotterli1 @Gabrielleling1 @Fairchildma2 @eastmanzhao @damarissun2 6p� RT @TheAndreG\_: " Kailangan ko munang mahalin ang sarili ko ngayon.. Masyado yata'ng napunta sayo ang lahat ng bagay na dapat para sa saril& RT @syahir\_sageng: Sepupu aku wedding invitations card, sketch sendiri dan scan. Dia arkitek https://t.co/rQTLLZbYUk RT @sorluciacaram: Nada justifica el odio, la obstinaci�n y el enfrentamiento entre pueblos diferentes o hermanos. La convivencia mere& RT @cxpucinepatouet: Le froid jveux bien mais la pluie nn c trop RT @IslamicTwee7s: One day you will leave this dunya, but your imprint will remain, so make it a good one. https://t.co/4Z8R7vjZQD RT @LelakiJelita: Sekarang dia buang kau macam kaca pecah but one day akan ada orang kutip kau macam permata pulak. Trust me we need to watch the flu, the flu 2 & world war z during our one week vacation next week so we'll understand the discussion and be able to answer the questions our health teacher will give us when we come back i live for this concept RT @hidayahaaaaah: @kaidorable114 @EXOVotingSquad @weareoneEXO Are u using incognito mode? Can u switch off phone then on back. \#EXO @weare& Jadikan kepandaian sebagai kebahagiaan bersama, sehingga mampu meningkatkan rasa ikhlas tuk bersyukur atas kesuksesa @DIVYAGARG11 @shivimomolove @RidaCreations They all know everything  @BrawlJo Si, et crois moi que je vais en avoir besoin. �\\\_(�)\_/� @laurensell @sparkycollier @zehicle @emaganap @ashinclouds @davidmedberry @TomFifield Be there shortly!! Esta noche en \#ATV, Alsurcanalsur har� un recorrido por la extensa y variada programaci�n del festivalsevilla de& https://t.co/OZYzf0Nb47 What a spectacular morning in the \#LakeDistrict! This is Yew Tree Tarn near \#Coniston right NOW! B  https://t.co/LyxjkuABD0 T� com vontade de pastelzin, mas se eu parar pra comer, vou me atrasar muito mais @AnnaRvr chaud T RT @CridaDemocracia: Quina casualitat. Cap d'aquets diaris obre portada amb la not�cia que ahir es va acreditar al Congreso que& didnt go to uni today and found out my groups been put in a seating plan, A SEATING PLAN!!! RT @antynatalizm: za 5 mld lat sBoDce zamieni si w czerwonego olbrzyma, a kilkaset mln lat p�zniej i tak pochBonie nasz planet, wic na& @BigChang\_ Anledning �r helt enkelt att ett innehav inte ska f� p�verka i h�gre utstr�ckning relativt �vriga andra. Tanken �r att alla innehav &gt; RT @ISzafranska: Do dopieri pierwsze powa|ne ostrze|enie. Strach pomy[le co mog zawiera kolejne. https://t.co/ScLLtcD6gC @LaurentM54 @CamilleTantale @GMeurice En cours.. Alf. .. �a veut dire ils sont en train de se branler ce qu'il leur reste de neurones , pas qu'ils ont r�ussi ! \#edf \#areva \#hulot RT @dalierzincanli: Oluna pet _i_e f1rlatan babaya hapis veren yarg1, bu _eref yoksunu magandalar1n d�rd�n� serbest b1rakt1. https://t.co/& @tera\_elin\_jp @yue\_Pfs �`JM Se cubrir�n las necesidades de empresarios afectados en la Av. Gobernadores y la R�a https://t.co/2B05BtMXPc \#NoticiasDC \#DiarioDeCampeche https://t.co/ATXBjzVl9q Texas church shooter Devin Kelley has a history of domestic violence https://t.co/fo0rMyv7g6 @redy\_eva\_re hahhaa, iyaa bang .. ntr aku peluk dgn erat smbil pejamkan mata, smpe kita kebumi, asyiiik ya bang ??   En Vlamir a la 1 Usted puede detectar el @YonGoicoechea falso, aprovechador, irreverente repecto de @VoluntadPopular, con estocolmo o sin �l. Iluso, t�ctico o estratega. Ud, decide. @Fundavade RT @heldbybronnor: Petition for @TheVampsJames to sing higher on \#NightAndDayTour rt to sign  RT @JeanetteEliz: @visconti1779 @TBGTNT @ky\_33 @Estrella51Ahora @moonloght\_22 @pouramours Thank you Meg. 
 Have a beautiful day every& my sister measured me dicc RT @CADCI: RECORDEU sempre que avui com el dia 3 els sindicats UGT i CCOO han fet costat a la patronal espanyola boicotejant& nude spy videos maria ozawa sex vids RT @ARnews1936: US Navy tests hypersonic missiles that can hit any target on Earth in under an hour https://t.co/Pesi1SjrJo via @IBTimesUK @NamiqKocharli Ve �ox maraql1d1 140 simvola nece catdirmisan bunu? The city was shining with the glory of God. It was shining bright like a very expensive jewel, like a jasper. It was clear as crystal. @eysertwo ayynaa,hahahhaha welcome beeb basta ikaw hahaha RT @JoyAnnReid: If Democrats DID take back the House, these ranking House members would become committee chairs ... with subpoena p& RT @HarryEStylesPH: ]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]]& RT @AfriqueTweet: 4 ��Panama Papers���: Dan Gertler, roi de la \#RDCongo et de lOffshore avec la B�n�diction de Joseph Kabila https://t.co/& @haries\_rajawali BO aja @i\_bobbyk Sampe mules ga oppa? RT @HugotDre: Gusto mo siyang tanungin about sa feelings niya kaya lang natatakot kang masaktan. RT @SaharaRayxXx: Demi Rose Mawby Flaunts Her BIG Boobs In A Deep Cleavage As She Wears A Tiny Bikini Top Enjoying Ibiza Sun https://t.co/3& RT @MassVotingArmy: New Tagline will be release later at 5PM. Stay tuned! Lets do this ARMYs! @czmoga @YouTube If you want to obtain good score then try \#PTEAcademic. For sample Papers, visit https://t.co/szRJZmSaAs. Visit here https://t.co/NigQNQuM7C Has she blamed her diary sec or researcher yet? \#Patel RT @aja\_dreamer23: "I don't have a sugar daddy. I've never had a sugar daddy. If I wanted a sugar daddy, yes, I could probably go out& @TenguOficial Joe, haber preguntado que si la vend�a y que por cu�nto, y sobretodo si la entrega con monos o esos van a parte  RT @bryanclemrj: Brant venceu por 250 sem contar a urna da disc�rdia. Olhem todas as urnas. Essa \#URNA7 � uma vergonha. Nunca a fam�& RT @MagazineCapital: Le Canard confirme les infos de Capital. En comptant les primes, elle aurait touch� 116.000 euros de salaires indus. h& @sportyfedz @delavinkisses @chinodelavin @imeecharlee @KissesNationPBB Wow all out support po. Ang gandang tignan. RT @SSatsaheb: @realDonaldTrump @POTUS @EdWGillespie \#@(G\_@\_0>9 Who is the Great Saint according to Nostradamus nd other foretel& RT @NiallOfficial: Cannot wait for tomorrow ! @MarenMorris and I are super excited . @CountryMusic RT @SamCoatesTimes: Will the Damian Green inquiry have concluded by the time decisions are announced about Priti Patel? https://t.co/xVcxy4& Tweet tweet. this is the first time in the history of me watching things that I saw two charas and brain just went  Also, it has snowed in parts of PA so no one better complain when I start singing Christmas music�d Girls appreciate even the small efforts. @mor1019 \#MORPinoyBiga10 Di Ko Lang Masabi by Kisses Delavin @KeirethLoL Le dijo la sart�n al cazo. RT @kuro\_chrome: What a wonderful motivation message -� Let's all vote guys, no matter what! And even if we don't manage to close it& Siempre hay que sentirse positivo � Apoyar una huelga convocada por terroristas y llamar fascistas a los que salen a la calle con la rojigualda. Logica indepe. @Real\_A\_Cullip1 My favourite was that South African rock dude O�O�O�O� @hatunturkish Kil yumagi amk az temiz ol temiz RT @umvesgo: - vc parece ser nervosa kkkk - kkkk sou nada sou tranquila https://t.co/76FPDgaQLu @NashScarecrow89 @adriaraexx @Brazzers S� RT @ObservadBinario: �Otro "sabotaje"? Nos van a dejar sin luz y sin agua en todo el pa�s... Ya estamos sin dinero y in comida ni medici& RT @Punziella: Water. Earth. Fire. Air. Long ago, the four nations lived together in harmony. Then, everything changed when the Fi& RT @ramblingsloa: Be yourself; Everyone else is already taken. Oscar Wilde https://t.co/3n2VZMCKp4 RT @Autosport\_Show: BREAKING: The 2018 @OfficialWRC season will be launched at \#ASI18! All competing WRC teams, drivers, co-drivers an& RT @kayafoo: wash your face, drink water, eat healthy, be humble, stay positive and continue grinding. RT @cristina\_pardo: El jefe de la UDEF dice que Rajoy cobr� en B y que G�rtel es "corrupci�n en estado puro", pero debatamos sobre la camis& RT @rebeccaschreibt: Dirk und Stephan sind \#obdachlos und suchen auf ungew�hnlichem Weg nach einer Bleibe. https://t.co/p8cg0wwN5l via @vic& @MarcherLord1 Haha  never heard so much fuss about absolutely 2 5ths of tit all RT @rHarry\_Styls: Here's a Harry Styles smiling appreciation tweet You're welcome https://t.co/uCnSy1olMI RT @ronflexzer: c'est fou comme le soir je remets toute ma vie en question RT @chadwickboseman: It's hard for a good man to be king...sat down with @CNET for an in-depth talk on all things \#BlackPanther, tech, a& RT @beitacabronis: -Cari�o , �por qu� vienes tan contenta? -Porque no sab�a que era multiorg�smica RT @UniofOxford: A clear vision for the future: can \#AR give sight back to 100 million people? @OxSightUK \#StartedInOxford& RT @ronflexzer: c'est fou comme le soir je remets toute ma vie en question RT @bretmanrock: If you guys see me in public and my nose is oily.... please let me know.+ I promise I won't think you're rude.. I'll hone& RT @jcmaningat: Yesterday: Underground women's group MAKIBAKA joined the lightning rally in Mendiola to commemorate the victory of& @rezaafathurizqi @karman\_mustamin Kalijodo izinnya sama preman", dan bnyak yg melanggar peraturan. RT @MannyMua733: Are the Dolan twins really only 17? Asking for a friend RT @XHSports: It's truely a fresh try for @andy\_murray dressed like this in a Glasgow charity match vs. @rogerfederer \#Federer. S& @RifantiFanti yaiyalahh.. Tb to DCs first night out in Leeds when he was home by 1.30 and sick in my bed ahahahaha https://t.co/yrGihYVIak @Alice\_Staniford @BikeShedDevon How do you find the disk brakes in the wet weather?? Sigo con 140 caracteres. Gracias, Twitter. \#TuitSerio RT @cdrsabadell: ��� Som una gentada i ens acompanya bona m�sica! Lluita i alegria que �s \#VagaGeneral8N!! \#CDRenXarxa& RT @bhogleharsha: Only 3 days left for \#InquizitiveMinds, India's biggest quiz contest. Catch me LIVE on Sunday, 12 Nov, in Bengaluru.�http& @5stars\_hanaebi �ܹkuU�WD @HiKONiCO Is it not okay to be white? Po tym wywiadzie nie przyjmuje juz zadnych argumentow broniacych decyzje Prezydenta.BASTA! https://t.co/e4dgkBWm2u RT @Green20William: Our CLUB pays it's own way.Pays all its bills.Step up anyone who can prove otherwiseM https://t.co/rVBc2Jw4mc RT @siirist: "Varl11n1n birileri i�in k1ymetli olu_u paha bi�ilemez. ��nk� senin tutunacak dal1n kalmasa bile sen birileri i�in bir dals1n& RT @EdMay\_0328: Ang baba ng views.. We can do better. \#MAYWARDMegaIGtorial @norimasa1914 W�h� @QSwearyLIII Its going to end in tears current mood: vampire princess  RT @\_amalves: Se at� Jesus n�o agradou todo mundo quem sou eu pra agradar n�o � mesmo, os meme t� a� quem gostou usa, quem n�o gostou paci�& NERV THIS! Hahahahahahahahahaa RT @OlivierGuitta: \#SaudiArabia government is aiming to confiscate cash and other assets worth as much as $800 billion in its broadeni& Dev\_Fadnavis: Its just one year and these huge numbers have started speaking already! Maharashtra salutes Hon PM & https://t.co/IU0VgVFU8L RT @JovenEuropeo: A ver c�mo conjug�is lo de la libertad y el "derecho a decididir" con todas las coacciones y piquetes que est�is ej& @vitortardin Falta de dar n� RT @andevaa: Oli pakko tehd� tilastoja Divarin 5-minuutin rangaistuksista, kun alkoi pist�m��n silm��n t�m� isojen j��hyjen suma& �� aaa 6� �0 tܑ�? \#REALFI Vilar NON p� hanen eller kommer det ytterligare v�g,,, minns pressen tidigare i Hamlet,,, https://t.co/cLNnmuOEui RT @SokoAnalyst: We are going to see firms folding up because whether we like it or not, consumers listen to Raila & the impact is building& Sad news this morning - little Banty is missing, presumed picked up by either a Fox, Stoat or Tawny. Had the freedom of the garden as the other chickens picked on her, but not in her shed at lock up last night. Hope she didn't suffer too much """ https://t.co/UTuYSeFXRM RT @pemarisanchez: Feliz de compartir este nuevo proyecto con tan maravillosos compa�eros, en la que ha sido tantas veces mi casa& RT @BangBangtan\_Esp: \[!\] Se confirma que BTS ser� invitado al programa "Jimmy Kimmel Live" mientras est�n en los Estados Unidos para la& stuck in this illusion called us, stuck in this puzzle and it doesn't get any better � RT @I\_CSC: � Recordeu: �Cap mena d'acci� violenta Ignoreu les provocacions Suport mutu �No difondre rumors LParticipeu de& @pvoorbach @JelmerGeerds Komt het dit seizoen nog goed met je club rooie vriend? Ik heb er persoonlijk een hard hoofd in. RT @CorreaDan: With govt funding plummeting, U.S. researchers need to find new ways to fund R&D - My article w/ @scott\_andes https://t.co& I just wish the sleep would come...knock me out so I feel nothing...\#insomnia @riethegreat\_ haboo ate oo create account, rpg game na sya and u get to socialize and stuff Ecovention Europe: Art to Transform Ecologies, 1957-2017 (part 2) Men always reach for technology, for development. They insist it will bring us to higher levels of progress. They havent the patience to work with slow-growing plants... WMMNA - https://t.co/NA3spa1g4j https://t.co/gWUG4CxZll
