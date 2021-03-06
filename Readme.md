# Text generation using Open AI's [GPT-2](https://github.com/openai/gpt-2) and hosting it in Google Cloud Run.

This project outlines tuning [OpenAI](https://github.com/openai/gpt-2) GPT-2 using Reddit comments and hosting the model app via Docker in Google Cloud.

## What is GPT-2? 

The OpenAI GPT-2 language model is a direct successor to GPT. GPT-2 has 1.5B parameters, 10x more than the original GPT, and it achieves SOTA results on 7 out of 8 tested language modeling datasets in a zero-shot transfer setting without any task-specific fine-tuning. The pre-training dataset contains 8 million Web pages collected by crawling qualified outbound links from Reddit.

I am using Max Woolf's [gpt-2-simple](https://github.com/minimaxir/gpt-2-simple) package that wraps existing model fine-tuning and generation scripts for OpenAI's GPT-2 text generation model (specifically the **"small"** 124M and **"medium"** 355M hyperparameter versions). The model finetuning is done in TensorFlow. 

## Where to get text data? 

First of all we need text data to finetuning GPT-2 for our application. It's up to you to look for data sources. Perhaps you already have a corpus of text? I decided to use Reddit comments hosted in BigQuery's reddit comments [database](https://console.cloud.google.com/bigquery?utm_source=bqui&utm_medium=link&utm_campaign=classic&project=charismatic-sum-134503). For fun, I wanted to generate YTA comments in [AITA](https://www.reddit.com/r/AmItheAsshole/) subreddit. You may have to setup your Google cloud credentials before you can begin querying. 

This is an example script to get comments from the subreddit. Feel free to change this to your needs. 

```
#standardSQL
SELECT
  REGEXP_REPLACE(REGEXP_REPLACE(REGEXP_REPLACE(REGEXP_REPLACE(body, '&amp;', '&'), '&lt;', '<'), '&gt;', '>'), '�', '') AS body
FROM
  `fh-bigquery.reddit_comments.2019_0*`
WHERE
  subreddit = 'AmItheAsshole'
  AND REGEXP_CONTAINS(body, r'(?i)YTA')
  AND author not in ('AutoModerator')
LIMIT
  100
```

You can save the data set locally or to your google drive.

![gbq_image](/images/gcp_save_data.PNG)

## GPT-2 fine-tuning

Use the accompanying colab [notebook](https://github.com/addadda023/GPT-2-text-generation/blob/master/Train_a_GPT_2_Text_Generating_Model.ipynb) in this repository to understand more. In summary:

* Use gpt-2-simple to load gpt-2.
* Load the input text file.
* Specify tuning parameters.
* Train and save the model.
* Generate texts.
* Save the TensorFlow checkpoint to Google drive and download locally. This is needed to deploy the model.

Notice the average loss of the model when training. It is easy to **overfit**. You can stop the training once you start to see it's decreasing slowly. 

## Making GPT-2 App

You have fine tuned GPT-2 using your text. That's great! But how do to host/deploy it? Many AI tutorials often show how to deploy a small model to a web service by using [Flask](https://www.fullstackpython.com/flask.html). The problem with GPT-2 is that it’s such a huge model that most conventional wisdom won't work well to get a performant app. Even if you do get it to run fast (e.g. by running the app on a GPU), it won’t be cheap, specially if there is a surge of requests.

One of the options for deployment is Google Cloud Run. Its a managed compute platform that automatically scales stateless containers. Cloud Run only charges for compute used *(effectively free)* and can scale indefinitely if there’s a traffic surge. Max Woolf has created a nice [package](https://github.com/minimaxir/gpt-2-cloud-run) to get started with this really easily. The end result is an app that can be used easily and cost-effectively to allow others to play with a finetuned GPT-2 model on another dataset, and allow programmatic access to the generated text. Check my sample [here](https://addadda023.github.io/GPT-2-text-generation/). Note that the deployed api itself doesn't provide a HTML frontend, but there are frontend templates to help you with that.

### Building a docker container

Since Cloud Run is stateless without access to local storage, you must bundle the model within the container. First, clone the [gpt-2-cloud-run](https://github.com/minimaxir/gpt-2-cloud-run) repo and copy the downloaded GPT-2 model into the folder (the model should be in the form of the folder hierarchy `/checkpoint/run1`. You can also clone this repo and find the same files in gpt-2-cloud-run folders.

**Before building the image**, edit the the dockerfile to specify which tensorflow version you want to install. It is recommended to use the same version you used for finetuning. You can find the version in colab using `gpt2.tf.__version__`

Then build the image:

```
docker build . -t gpt2
```

If you want to test the image locally with the same specs as Cloud Run, you can run:

```
docker run -p 8080:8080 --memory="2g" --cpus="1" gpt2
```

If it runs successfully you can then visit/curl `http://0.0.0.0:8080` to get generated text! Note that 0.0.0.0:8080 is also the port the Google cloud container listens to when deployed.

### Upload the image to Google Container Registry

You want to deploy the image in the cloud, but you must first upload the image to [container registry](https://console.cloud.google.com/gcr). 

Tag the image and upload it to the Google Container Registry (note, this will take awhile due to the image size!). Also note you have to setup your credentials which involves steps you will be notified when you execute below commands. **What is PROJECT-ID?** It's the name of the project you will create in GCR before uploading the image.

```
docker tag gpt2 gcr.io/[PROJECT-ID]/gpt2
docker push gcr.io/[PROJECT-ID]/gpt2
```

The end result should look like this:

![image_of_gcr](https://github.com/addadda023/GPT-2-text-generation/blob/master/images/gcr_sample.png)

### Deploy the container

Once the image is uploaded you can head to [Google Cloud Console](https://console.cloud.google.com/run?project=yta-gpt2&folder=&organizationId=) to deploy it. Click on **Create Service** and follow the instructions. **Remember to set Memory Allocated to 2 GB and Maximum Requests Per Container to 1!**. You should see a success prompt shortly. You can explore the console further yourself.

![image_of_container](https://github.com/addadda023/GPT-2-text-generation/blob/master/images/Deployed_container_sample.png)

Congratulations! You just deployed the model. 

### Interacting with the api

The api serves GET & POST requests over HTTP and returns a JSON object with a text attribute that contains the generated text. Say the cloud run app url is `comment-generator.run.app`. A `GET` request to the API would be `http://comment-generator.run.app?length=100&temperature=1.0` which can be accessed by almost any type of client. Go ahead and try in your client or browser.

A `POST` request (passing the data as a JSON object) is more ideal as it is both more secure and allows non-ASCII inputs. 

Python example:
```
import requests

req = requests.post('http://comment-generator.run.app',
                    json={'length': 100, 'temperature': 1.0})
text = req.json()['text']
print(text)
```
Feel free to modify the `app_ui.html` file provided in the docker repository to suit your needs. It utilizes AJAX POST requests via jQuery to retrieve the generated text and parse the data for display. Check my example [here](https://addadda023.github.io/GPT-2-text-generation/).

## Some notes

* Due to Cloud Run's current 2 GB memory maximum, this app will only work with the 117M "small" GPT-2 model, and not the 345M "medium" model (even if Cloud Run offers a 4 GB option in the future, it would not be enough to support the 345M model).
* Each prediction, at the default 1023 token length, will take about 2 minutes to generate (10 seconds per 100 tokens). You may want to consider reducing the length of the generated text if speed is a concern and/or hardcapping the length at the app-level.
* If your API on Cloud Run is actively processing a request less than 7% of the time (at the 100 millisecond level) in a given month, you'll stay within the free tier of Cloud Run, and the price is $0.10 an hour if the service goes over the free tier. Only the time starting up an instance and processing a request counts as billable time (i.e. the durations in the logs); idle time does not count as billable, making it surprisingly easy to stay within the limits.
* The concurrency is set to 1 to ensure maximum utilization for each user (if a single user is using it and accidently causes another container to spawn, it doesn't matter cost-wise as only requests processing incurs charges, not the number of active containers).
* Memory leaks in the container may cause you to go over the 2GB limit and crash the container after enough text generations. Fortunately, Cloud Run can quickly recover (although the current request will fail), and having multiple containers operating due to low concurrency can distribute the workload.
