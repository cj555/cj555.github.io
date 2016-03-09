---
layout: post
title:  "Real-time Machine Learning"
date:   2016-03-08 12:15:56 +0800
categories: Tech

---

## - A demo of random Forest classification on Storm


### Purpose

The arthur in this [article](PMML Support in Spark MLlib) states:


> Model building is a complex task, it is performed on a large amount of historical data and requires a fast and scalable engine to produce correct results: this is where Apache Spark’s MLlib shines.



Therefore, separating the modelling and scoring process might be more efficient.


### Experiemnt

Here we conduct a simple experiment that running a random forest model in storm. We fit a random forest model in R and export the PMML file. Then the pmml is loaded into storm topology. Such topology is able to do the realtime classification upon spout output.    



1. Produce a pmml file
First, train a random forest model using R code.

```R

  library("pmml")
  library("randomForest")

  ## view xml @ http://xmlgrid.net/

  iris.randomForest = randomForest(Species ~ ., iris, ntree = 5, maxnodes = 2)

  saveXML(pmml(iris.randomForest), "D:/workspace/infotrie/spark_storm/pmml_demo/RandomForestIris.pmml")

```

2. Run classification on Storm

PMML file contains all the information necessary to rebuild the machine learning model. Here we demonstrate a simple `random forest` model with 5 trees, 4 features and 1 targeted value. The `jpmml` package can convert the pmml file back to a machine learning model. After conversion, the machine learning model will stay in the memory and response once a time upon the output of storm spout.


```java

package pmml;

import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.InputStream;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Random;

import backtype.storm.Config;
import backtype.storm.LocalCluster;
import backtype.storm.StormSubmitter;
import backtype.storm.spout.SpoutOutputCollector;
import backtype.storm.task.OutputCollector;
import backtype.storm.task.TopologyContext;
import backtype.storm.topology.OutputFieldsDeclarer;
import backtype.storm.topology.TopologyBuilder;
import backtype.storm.topology.base.BaseRichBolt;
import backtype.storm.topology.base.BaseRichSpout;
import backtype.storm.tuple.Fields;
import backtype.storm.tuple.Tuple;
import backtype.storm.tuple.Values;
import backtype.storm.utils.Utils;
import java.util.concurrent.ThreadLocalRandom;

import javax.xml.bind.JAXBException;
import javax.xml.transform.Source;

import org.dmg.pmml.FieldName;
import org.dmg.pmml.PMML;
import org.jpmml.evaluator.Evaluator;
import org.jpmml.evaluator.FieldValue;
import org.jpmml.evaluator.ModelEvaluator;
import org.jpmml.evaluator.ModelEvaluatorFactory;
import org.jpmml.model.ImportFilter;
import org.jpmml.model.JAXBUtil;
import org.xml.sax.InputSource;
import org.xml.sax.SAXException;

/**
 * This is a basic example of a Storm topology.
 */
public class PmmlDemoTopology {

	public static class RandomSentenceSpout extends BaseRichSpout {
		SpoutOutputCollector _collector;
		Random _rand;

		public void open(Map conf, TopologyContext context, SpoutOutputCollector collector) {
			_collector = collector;
			_rand = new Random();
		}


		public void nextTuple() {

			Utils.sleep(100);
			int[] solutionArray = { 1, 2, 3, 4 };
			shuffleArray(solutionArray);
			String[] sentences = new String[] { "1,2,3,4", "1,3,3,4", "5,3,3,4", "1,3,6,4" };

			String sentence = sentences[_rand.nextInt(sentences.length)];
			_collector.emit(new Values(sentence));

		}

		@Override
		public void ack(Object id) {
		}

		@Override
		public void fail(Object id) {
		}

		@Override
		public void declareOutputFields(OutputFieldsDeclarer declarer) {
			declarer.declare(new Fields("word"));
		}

	}

	public static class RandomForestBolt extends BaseRichBolt {

		OutputCollector _collector;
		Evaluator _evaluator;

		public void prepare(Map conf, TopologyContext context, OutputCollector collector) {
			_collector = collector;
			// _evaluator = (Evaluator) conf.get("rf.evaluator");
			InputStream is;
			try {
				is = new FileInputStream(
						"RandomForestIris.pmml");
				Source transformedSource = ImportFilter.apply(new InputSource(is));
				PMML pmml;

				pmml = JAXBUtil.unmarshalPMML(transformedSource);

				TopologyBuilder builder = new TopologyBuilder();

				ModelEvaluatorFactory modelEvaluatorFactory = ModelEvaluatorFactory.newInstance();

				ModelEvaluator<?> modelEvaluator = modelEvaluatorFactory.newModelManager(pmml);

				_evaluator = (Evaluator) modelEvaluator;

			} catch (FileNotFoundException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			} catch (SAXException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			} catch (JAXBException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}

		}

		public void execute(Tuple tuple) {

			String[] inputs = tuple.getString(0).split(",");

			List<FieldName> activeFields = _evaluator.getActiveFields();
			List<FieldName> targetFields = _evaluator.getTargetFields();
			List<FieldName> outputFields = _evaluator.getOutputFields();

			Map<FieldName, FieldValue> arguments = new LinkedHashMap<FieldName, FieldValue>();
			int i = 0;
			for (FieldName activeField : activeFields) {
				// The raw (ie. user-supplied) value could be any Java primitive
				// value

				Object rawValue = Double.parseDouble(inputs[i]);
				System.out.println(activeField + ": " + rawValue);
				i++;

				// The raw value is passed through: 1) outlier treatment, 2)
				// missing value treatment, 3) invalid value treatment and 4)
				// type conversion
				FieldValue activeValue = _evaluator.prepare(activeField, rawValue);

				arguments.put(activeField, activeValue);
			}

			Map<FieldName, ?> results = _evaluator.evaluate(arguments);
			FieldName targetName = _evaluator.getTargetField();
			Object targetValue = results.get(targetName);

			System.out.println(targetValue);

			_collector.emit(tuple, new Values(targetValue));
			_collector.ack(tuple);
		}

		@Override
		public void declareOutputFields(OutputFieldsDeclarer declarer) {
			declarer.declare(new Fields("word"));
		}

	}

	public static void main(String[] args) throws Exception {
		TopologyBuilder builder = new TopologyBuilder();

		builder.setSpout("word", new RandomSentenceSpout(), 2);
		builder.setBolt("exclaim1", new RandomForestBolt(), 3).shuffleGrouping("word");

		Config conf = new Config();
		conf.setDebug(true);

		if (args != null && args.length > 0) {

			conf.setNumWorkers(3);
			StormSubmitter.submitTopologyWithProgressBar(args[0], conf, builder.createTopology());

		} else {

			LocalCluster cluster = new LocalCluster();
			cluster.submitTopology("test", conf, builder.createTopology());
			Utils.sleep(10000);
			cluster.killTopology("test");
			cluster.shutdown();
		}

	}}

```




###Current Status


`pmml` package in R can convert most of ml models into pmml format. Python world is little messy, which have a lot like `pypmml`. For now, spark only provides a few APIs to export PMML file. However, PMML is not something strange but a XML file with a lot definitions of ml models. So it's possible to config one like [here](http://dmg.org/pmml/v4-1/NaiveBayes.html)


### Summary

It's possible to have once-a-time or say real-time machine learning engine using the models from spark. And if such model is small enough to stay in the memory, very low latency can be achieved.
