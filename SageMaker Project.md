# Creating a Sentiment Analysis Web App
## Using PyTorch and SageMaker

_Deep Learning Nanodegree Program | Deployment_

---

Now that we have a basic understanding of how SageMaker works we will try to use it to construct a complete project from end to end. Our goal will be to have a simple web page which a user can use to enter a movie review. The web page will then send the review off to our deployed model which will predict the sentiment of the entered review.

## Instructions

Some template code has already been provided for you, and you will need to implement additional functionality to successfully complete this notebook. You will not need to modify the included code beyond what is requested. Sections that begin with '**TODO**' in the header indicate that you need to complete or implement some portion within them. Instructions will be provided for each section and the specifics of the implementation are marked in the code block with a `# TODO: ...` comment. Please be sure to read the instructions carefully!

In addition to implementing code, there will be questions for you to answer which relate to the task and your implementation. Each section where you will answer a question is preceded by a '**Question:**' header. Carefully read each question and provide your answer below the '**Answer:**' header by editing the Markdown cell.

> **Note**: Code and Markdown cells can be executed using the **Shift+Enter** keyboard shortcut. In addition, a cell can be edited by typically clicking it (double-click for Markdown cells) or by pressing **Enter** while it is highlighted.

## General Outline

Recall the general outline for SageMaker projects using a notebook instance.

1. Download or otherwise retrieve the data.
2. Process / Prepare the data.
3. Upload the processed data to S3.
4. Train a chosen model.
5. Test the trained model (typically using a batch transform job).
6. Deploy the trained model.
7. Use the deployed model.

For this project, you will be following the steps in the general outline with some modifications. 

First, you will not be testing the model in its own step. You will still be testing the model, however, you will do it by deploying your model and then using the deployed model by sending the test data to it. One of the reasons for doing this is so that you can make sure that your deployed model is working correctly before moving forward.

In addition, you will deploy and use your trained model a second time. In the second iteration you will customize the way that your trained model is deployed by including some of your own code. In addition, your newly deployed model will be used in the sentiment analysis web app.

## Step 1: Downloading the data

As in the XGBoost in SageMaker notebook, we will be using the [IMDb dataset](http://ai.stanford.edu/~amaas/data/sentiment/)

> Maas, Andrew L., et al. [Learning Word Vectors for Sentiment Analysis](http://ai.stanford.edu/~amaas/data/sentiment/). In _Proceedings of the 49th Annual Meeting of the Association for Computational Linguistics: Human Language Technologies_. Association for Computational Linguistics, 2011.


```python
%mkdir ../data
!wget -O ../data/aclImdb_v1.tar.gz http://ai.stanford.edu/~amaas/data/sentiment/aclImdb_v1.tar.gz
!tar -zxf ../data/aclImdb_v1.tar.gz -C ../data
```

    mkdir: cannot create directory ‘../data’: File exists
    --2020-09-10 18:25:34--  http://ai.stanford.edu/~amaas/data/sentiment/aclImdb_v1.tar.gz
    Resolving ai.stanford.edu (ai.stanford.edu)... 171.64.68.10
    Connecting to ai.stanford.edu (ai.stanford.edu)|171.64.68.10|:80... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 84125825 (80M) [application/x-gzip]
    Saving to: ‘../data/aclImdb_v1.tar.gz’
    
    ../data/aclImdb_v1. 100%[===================>]  80.23M  25.3MB/s    in 3.6s    
    
    2020-09-10 18:25:37 (22.2 MB/s) - ‘../data/aclImdb_v1.tar.gz’ saved [84125825/84125825]
    


## Step 2: Preparing and Processing the data

Also, as in the XGBoost notebook, we will be doing some initial data processing. The first few steps are the same as in the XGBoost example. To begin with, we will read in each of the reviews and combine them into a single input structure. Then, we will split the dataset into a training set and a testing set.


```python
import os
import glob

def read_imdb_data(data_dir='../data/aclImdb'):
    data = {}
    labels = {}
    
    for data_type in ['train', 'test']:
        data[data_type] = {}
        labels[data_type] = {}
        
        for sentiment in ['pos', 'neg']:
            data[data_type][sentiment] = []
            labels[data_type][sentiment] = []
            
            path = os.path.join(data_dir, data_type, sentiment, '*.txt')
            files = glob.glob(path)
            
            for f in files:
                with open(f) as review:
                    data[data_type][sentiment].append(review.read())
                    # Here we represent a positive review by '1' and a negative review by '0'
                    labels[data_type][sentiment].append(1 if sentiment == 'pos' else 0)
                    
            assert len(data[data_type][sentiment]) == len(labels[data_type][sentiment]), \
                    "{}/{} data size does not match labels size".format(data_type, sentiment)
                
    return data, labels
```


```python
data, labels = read_imdb_data()
print("IMDB reviews: train = {} pos / {} neg, test = {} pos / {} neg".format(
            len(data['train']['pos']), len(data['train']['neg']),
            len(data['test']['pos']), len(data['test']['neg'])))
```

    IMDB reviews: train = 12500 pos / 12500 neg, test = 12500 pos / 12500 neg


Now that we've read the raw training and testing data from the downloaded dataset, we will combine the positive and negative reviews and shuffle the resulting records.


```python
from sklearn.utils import shuffle

def prepare_imdb_data(data, labels):
    """Prepare training and test sets from IMDb movie reviews."""
    
    #Combine positive and negative reviews and labels
    data_train = data['train']['pos'] + data['train']['neg']
    data_test = data['test']['pos'] + data['test']['neg']
    labels_train = labels['train']['pos'] + labels['train']['neg']
    labels_test = labels['test']['pos'] + labels['test']['neg']
    
    #Shuffle reviews and corresponding labels within training and test sets
    data_train, labels_train = shuffle(data_train, labels_train)
    data_test, labels_test = shuffle(data_test, labels_test)
    
    # Return a unified training data, test data, training labels, test labets
    return data_train, data_test, labels_train, labels_test
```


```python
train_X, test_X, train_y, test_y = prepare_imdb_data(data, labels)
print("IMDb reviews (combined): train = {}, test = {}".format(len(train_X), len(test_X)))
```

    IMDb reviews (combined): train = 25000, test = 25000


Now that we have our training and testing sets unified and prepared, we should do a quick check and see an example of the data our model will be trained on. This is generally a good idea as it allows you to see how each of the further processing steps affects the reviews and it also ensures that the data has been loaded correctly.


```python
print(train_X[100])
print(train_y[100])
```

    I read John Everingham's story years ago in Reader's Digest, and I remember thinking what a great movie it would make. And it probably would have been had Michael Landon never got his hands on it. As far as I'm concerned, Landon was one of the worst actors on earth, and his artistic license went way over the top, similar to his massacre of the "Little House" book series is proof. The acting, for lack of a better word, is atrocious, the screenplay sloppy, and there are more close-ups of Landon's puss than should be allowed.<br /><br />This movie reflects Everingham's story as much as "Little House On The Prairie" reflects the books is was "based" on. It's just another vehicle to show off Landons horrendous hair.
    0


The first step in processing the reviews is to make sure that any html tags that appear should be removed. In addition we wish to tokenize our input, that way words such as *entertained* and *entertaining* are considered the same with regard to sentiment analysis.


```python
import nltk
from nltk.corpus import stopwords
from nltk.stem.porter import *

import re
from bs4 import BeautifulSoup

def review_to_words(review):
    nltk.download("stopwords", quiet=True)
    stemmer = PorterStemmer()
    
    text = BeautifulSoup(review, "html.parser").get_text() # Remove HTML tags
    text = re.sub(r"[^a-zA-Z0-9]", " ", text.lower()) # Convert to lower case
    words = text.split() # Split string into words
    words = [w for w in words if w not in stopwords.words("english")] # Remove stopwords
    words = [PorterStemmer().stem(w) for w in words] # stem
    
    return words
```

The `review_to_words` method defined above uses `BeautifulSoup` to remove any html tags that appear and uses the `nltk` package to tokenize the reviews. As a check to ensure we know how everything is working, try applying `review_to_words` to one of the reviews in the training set.


```python
# TODO: Apply review_to_words to a review (train_X[100] or any other review)
review_to_words(train_X[100])
```




    ['read',
     'john',
     'everingham',
     'stori',
     'year',
     'ago',
     'reader',
     'digest',
     'rememb',
     'think',
     'great',
     'movi',
     'would',
     'make',
     'probabl',
     'would',
     'michael',
     'landon',
     'never',
     'got',
     'hand',
     'far',
     'concern',
     'landon',
     'one',
     'worst',
     'actor',
     'earth',
     'artist',
     'licens',
     'went',
     'way',
     'top',
     'similar',
     'massacr',
     'littl',
     'hous',
     'book',
     'seri',
     'proof',
     'act',
     'lack',
     'better',
     'word',
     'atroci',
     'screenplay',
     'sloppi',
     'close',
     'up',
     'landon',
     'puss',
     'allow',
     'movi',
     'reflect',
     'everingham',
     'stori',
     'much',
     'littl',
     'hous',
     'prairi',
     'reflect',
     'book',
     'base',
     'anoth',
     'vehicl',
     'show',
     'landon',
     'horrend',
     'hair']



**Question:** Above we mentioned that `review_to_words` method removes html formatting and allows us to tokenize the words found in a review, for example, converting *entertained* and *entertaining* into *entertain* so that they are treated as though they are the same word. What else, if anything, does this method do to the input?

**Answer:** The method allows the content to be broken down into a series of bag of words by tokenizing, lemminizing and splitting the words and sentences. The algorithms also remove the stopwords to make the data more refined and eliminate any unnecessary terms from the entire content dataset by finally stemming the words and key phrases.

The method below applies the `review_to_words` method to each of the reviews in the training and testing datasets. In addition it caches the results. This is because performing this processing step can take a long time. This way if you are unable to complete the notebook in the current session, you can come back without needing to process the data a second time.


```python
import pickle

cache_dir = os.path.join("../cache", "sentiment_analysis")  # where to store cache files
os.makedirs(cache_dir, exist_ok=True)  # ensure cache directory exists

def preprocess_data(data_train, data_test, labels_train, labels_test,
                    cache_dir=cache_dir, cache_file="preprocessed_data.pkl"):
    """Convert each review to words; read from cache if available."""

    # If cache_file is not None, try to read from it first
    cache_data = None
    if cache_file is not None:
        try:
            with open(os.path.join(cache_dir, cache_file), "rb") as f:
                cache_data = pickle.load(f)
            print("Read preprocessed data from cache file:", cache_file)
        except:
            pass  # unable to read from cache, but that's okay
    
    # If cache is missing, then do the heavy lifting
    if cache_data is None:
        # Preprocess training and test data to obtain words for each review
        #words_train = list(map(review_to_words, data_train))
        #words_test = list(map(review_to_words, data_test))
        words_train = [review_to_words(review) for review in data_train]
        words_test = [review_to_words(review) for review in data_test]
        
        # Write to cache file for future runs
        if cache_file is not None:
            cache_data = dict(words_train=words_train, words_test=words_test,
                              labels_train=labels_train, labels_test=labels_test)
            with open(os.path.join(cache_dir, cache_file), "wb") as f:
                pickle.dump(cache_data, f)
            print("Wrote preprocessed data to cache file:", cache_file)
    else:
        # Unpack data loaded from cache file
        words_train, words_test, labels_train, labels_test = (cache_data['words_train'],
                cache_data['words_test'], cache_data['labels_train'], cache_data['labels_test'])
    
    return words_train, words_test, labels_train, labels_test
```


```python
# Preprocess data
train_X, test_X, train_y, test_y = preprocess_data(train_X, test_X, train_y, test_y)
```

    Read preprocessed data from cache file: preprocessed_data.pkl


## Transform the data

In the XGBoost notebook we transformed the data from its word representation to a bag-of-words feature representation. For the model we are going to construct in this notebook we will construct a feature representation which is very similar. To start, we will represent each word as an integer. Of course, some of the words that appear in the reviews occur very infrequently and so likely don't contain much information for the purposes of sentiment analysis. The way we will deal with this problem is that we will fix the size of our working vocabulary and we will only include the words that appear most frequently. We will then combine all of the infrequent words into a single category and, in our case, we will label it as `1`.

Since we will be using a recurrent neural network, it will be convenient if the length of each review is the same. To do this, we will fix a size for our reviews and then pad short reviews with the category 'no word' (which we will label `0`) and truncate long reviews.

### (TODO) Create a word dictionary

To begin with, we need to construct a way to map words that appear in the reviews to integers. Here we fix the size of our vocabulary (including the 'no word' and 'infrequent' categories) to be `5000` but you may wish to change this to see how it affects the model.

> **TODO:** Complete the implementation for the `build_dict()` method below. Note that even though the vocab_size is set to `5000`, we only want to construct a mapping for the most frequently appearing `4998` words. This is because we want to reserve the special labels `0` for 'no word' and `1` for 'infrequent word'.


```python
import numpy as np

def build_dict(data, vocab_size = 5000):
    """Construct and return a dictionary mapping each of the most frequently appearing words to a unique integer."""
    
    # TODO: Determine how often each word appears in `data`. Note that `data` is a list of sentences and that a
    #       sentence is a list of words.
    
    word_count = {} # A dict storing the words that appear in the reviews along with how often they occur
    
    for review in data:
        for word in review:
            if word in word_count:
                word_count[word] += 1
            else:
                word_count[word] = 1
                
                
    # TODO: Sort the words found in `data` so that sorted_words[0] is the most frequently appearing word and
    #       sorted_words[-1] is the least frequently appearing word.
    
    sorted_words = [item[0] for item in sorted(word_count.items(), key = lambda x: x[1], reverse = True)]
    

    
    word_dict = {} # This is what we are building, a dictionary that translates words into integers
    for idx, word in enumerate(sorted_words[:vocab_size - 2]): # The -2 is so that we save room for the 'no word'
        word_dict[word] = idx + 2                              # 'infrequent' labels
        
    return word_dict
```


```python
word_dict = build_dict(train_X)
```

**Question:** What are the five most frequently appearing (tokenized) words in the training set? Does it makes sense that these words appear frequently in the training set?

**Answer:**


```python
# TODO: Use this space to determine the five most frequently appearing words in the training set.
print('Five most frequently appearing words in the training set are:\n')
print(sorted(word_dict, key = word_dict.get, reverse = False)[:5])
```

    Five most frequently appearing words in the training set are:
    
    ['movi', 'film', 'one', 'like', 'time']


### Save `word_dict`

Later on when we construct an endpoint which processes a submitted review we will need to make use of the `word_dict` which we have created. As such, we will save it to a file now for future use.


```python
data_dir = '../data/pytorch' # The folder we will use for storing data
if not os.path.exists(data_dir): # Make sure that the folder exists
    os.makedirs(data_dir)
```


```python
with open(os.path.join(data_dir, 'word_dict.pkl'), "wb") as f:
    pickle.dump(word_dict, f)
```

### Transform the reviews

Now that we have our word dictionary which allows us to transform the words appearing in the reviews into integers, it is time to make use of it and convert our reviews to their integer sequence representation, making sure to pad or truncate to a fixed length, which in our case is `500`.


```python
def convert_and_pad(word_dict, sentence, pad=500):
    NOWORD = 0 # We will use 0 to represent the 'no word' category
    INFREQ = 1 # and we use 1 to represent the infrequent words, i.e., words not appearing in word_dict
    
    working_sentence = [NOWORD] * pad
    
    for word_index, word in enumerate(sentence[:pad]):
        if word in word_dict:
            working_sentence[word_index] = word_dict[word]
        else:
            working_sentence[word_index] = INFREQ
            
    return working_sentence, min(len(sentence), pad)

def convert_and_pad_data(word_dict, data, pad=500):
    result = []
    lengths = []
    
    for sentence in data:
        converted, leng = convert_and_pad(word_dict, sentence, pad)
        result.append(converted)
        lengths.append(leng)
        
    return np.array(result), np.array(lengths)
```


```python
train_X, train_X_len = convert_and_pad_data(word_dict, train_X)
test_X, test_X_len = convert_and_pad_data(word_dict, test_X)
```

As a quick check to make sure that things are working as intended, check to see what one of the reviews in the training set looks like after having been processeed. Does this look reasonable? What is the length of a review in the training set?


```python
# Use this cell to examine one of the processed reviews to make sure everything is working as intended.
print('length of a review ', len(train_X[500]))
print(train_X[500])
```

    length of a review  500
    [   1 1120  578  238 1598  362  340    6 1011 1008   61    1 1771  477
     3028  104  373   97 2477  315    1  749  152  459  445   65 1385 2265
      149  291 1294  112  316 1729   27 1837  394  386 1655 1063 4009    1
     4128  315   97  570 1977    1  222   46   18 3658  358  296  495  196
     1154  627 4408    1 1142 2070 1310 4302  597 2657   10  851  801  910
        1  107  162  157 1650 1358  659   80  690    1   65 1125 1209 3010
      597    1    8 2313    4 2203 3028    1 1576  592   98 1124  755  945
        1   42 2694 1161  214 1650  760    1    4 1161  472 3679   64 3010
      561    1 2028    1 1058  489 1473 1620 3386    1    1    1  531 2694
      214 1650 1241   63 3738    1 3028  477  471 2144    1 3386  167    1
      167 4848    1 2907  303 2683 2106 3866 2151   40    8  705 2038  284
     1030 3386   11 1310  157  705  997  548   55  122  917 1195   42  970
     2115 1310  765 3386  574    1  133    1 3386  517 1027 1577  863 1733
     1170 1310    1 2768  629 1338 3386 2046 1599    1    1 3010  471 1650
       61 3277    1 4798 1650  609 4848 1310 2368 1310  469  416 3386   61
     1650 1033  273 1578    1  910  200 3087 1310 1646    1 2180 1616    1
      133 3010 1310 1646 3010 1126   57  648  471 3087    1  214   14 1310
     1650  750    1  350  419  152 3255  442 2258  593  489  451  128  450
      115 2970  406 3028 1132  939   97  104  840 1310 1650 4848    9 2424
     4733   92  370  213 1970 2083 2503 2481  271 1080 4848   40  489  911
     1852 1599   23  979   27   23   37    4   36 1224  981    6  185    1
        1   27 2969  383  773   72 3018  175  320  200  442  466    1 4158
     4820  114  915 1713   93 1592  152  177 1260  911 3070  930 1553 3481
      172    4 2390 1453  598 2016    1 2907   60  391 2189  553 1239 1964
       81 1310  972 1011  888  687  404 1978  211    6  421   75 1655 2747
     2330   49 1978  411   59  472 1011 1771  486 1003 2823 4848 2324    1
      231   14 3629 4848 1018  399    1  555  478 1650 1125 4553  201  568
     1771 1255  140 4038  435    1 4472    1  383 2897  128 1219 2657  659
      313   81    5 2907  286  149    1  127   26  315 1978    0    0    0
        0    0    0    0    0    0    0    0    0    0    0    0    0    0
        0    0    0    0    0    0    0    0    0    0    0    0    0    0
        0    0    0    0    0    0    0    0    0    0    0    0    0    0
        0    0    0    0    0    0    0    0    0    0    0    0    0    0
        0    0    0    0    0    0    0    0    0    0    0    0    0    0
        0    0    0    0    0    0    0    0    0    0    0    0    0    0
        0    0    0    0    0    0    0    0    0    0]


**Question:** In the cells above we use the `preprocess_data` and `convert_and_pad_data` methods to process both the training and testing set. Why or why not might this be a problem?

**Answer:**
The preprocess_data method helps in saving the outputs to a file and can reduce the overall time for getting the data back after the notebook is restarted. But it can take a long time for preprocessing if the data is highly unclean and requires more stringent tranformations. It may also be time consuming if the data is highly unstructured or filled with stopwords, leaving very little information to be processed and analyzed for sentiments. 
The convert_and_pad_data method helps in creating a constant length for all the reviews through the LSTM classifier which needs a constant batch size. 
One particular problem can arise from making all the vectors the same size as there is a considerable increase in the memory usage for the system, since it may not be necessary to keep a constant length for the reviews, given their non-uniform shape and size.

## Step 3: Upload the data to S3

As in the XGBoost notebook, we will need to upload the training dataset to S3 in order for our training code to access it. For now we will save it locally and we will upload to S3 later on.

### Save the processed training dataset locally

It is important to note the format of the data that we are saving as we will need to know it when we write the training code. In our case, each row of the dataset has the form `label`, `length`, `review[500]` where `review[500]` is a sequence of `500` integers representing the words in the review.


```python
import pandas as pd
    
pd.concat([pd.DataFrame(train_y), pd.DataFrame(train_X_len), pd.DataFrame(train_X)], axis=1) \
        .to_csv(os.path.join(data_dir, 'train.csv'), header=False, index=False)
```

### Uploading the training data


Next, we need to upload the training data to the SageMaker default S3 bucket so that we can provide access to it while training our model.


```python
import sagemaker

sagemaker_session = sagemaker.Session()

bucket = sagemaker_session.default_bucket()
prefix = 'sagemaker/sentiment_rnn'

role = sagemaker.get_execution_role()
```


```python
input_data = sagemaker_session.upload_data(path=data_dir, bucket=bucket, key_prefix=prefix)
```

**NOTE:** The cell above uploads the entire contents of our data directory. This includes the `word_dict.pkl` file. This is fortunate as we will need this later on when we create an endpoint that accepts an arbitrary review. For now, we will just take note of the fact that it resides in the data directory (and so also in the S3 training bucket) and that we will need to make sure it gets saved in the model directory.

## Step 4: Build and Train the PyTorch Model

In the XGBoost notebook we discussed what a model is in the SageMaker framework. In particular, a model comprises three objects

 - Model Artifacts,
 - Training Code, and
 - Inference Code,
 
each of which interact with one another. In the XGBoost example we used training and inference code that was provided by Amazon. Here we will still be using containers provided by Amazon with the added benefit of being able to include our own custom code.

We will start by implementing our own neural network in PyTorch along with a training script. For the purposes of this project we have provided the necessary model object in the `model.py` file, inside of the `train` folder. You can see the provided implementation by running the cell below.


```python
!pygmentize train/model.py
```

    [34mimport[39;49;00m [04m[36mtorch[39;49;00m[04m[36m.[39;49;00m[04m[36mnn[39;49;00m [34mas[39;49;00m [04m[36mnn[39;49;00m
    
    [34mclass[39;49;00m [04m[32mLSTMClassifier[39;49;00m(nn.Module):
        [33m"""[39;49;00m
    [33m    This is the simple RNN model we will be using to perform Sentiment Analysis.[39;49;00m
    [33m    """[39;49;00m
    
        [34mdef[39;49;00m [32m__init__[39;49;00m([36mself[39;49;00m, embedding_dim, hidden_dim, vocab_size):
            [33m"""[39;49;00m
    [33m        Initialize the model by settingg up the various layers.[39;49;00m
    [33m        """[39;49;00m
            [36msuper[39;49;00m(LSTMClassifier, [36mself[39;49;00m).[32m__init__[39;49;00m()
    
            [36mself[39;49;00m.embedding = nn.Embedding(vocab_size, embedding_dim, padding_idx=[34m0[39;49;00m)
            [36mself[39;49;00m.lstm = nn.LSTM(embedding_dim, hidden_dim)
            [36mself[39;49;00m.dense = nn.Linear(in_features=hidden_dim, out_features=[34m1[39;49;00m)
            [36mself[39;49;00m.sig = nn.Sigmoid()
            
            [36mself[39;49;00m.word_dict = [34mNone[39;49;00m
    
        [34mdef[39;49;00m [32mforward[39;49;00m([36mself[39;49;00m, x):
            [33m"""[39;49;00m
    [33m        Perform a forward pass of our model on some input.[39;49;00m
    [33m        """[39;49;00m
            x = x.t()
            lengths = x[[34m0[39;49;00m,:]
            reviews = x[[34m1[39;49;00m:,:]
            embeds = [36mself[39;49;00m.embedding(reviews)
            lstm_out, _ = [36mself[39;49;00m.lstm(embeds)
            out = [36mself[39;49;00m.dense(lstm_out)
            out = out[lengths - [34m1[39;49;00m, [36mrange[39;49;00m([36mlen[39;49;00m(lengths))]
            [34mreturn[39;49;00m [36mself[39;49;00m.sig(out.squeeze())


The important takeaway from the implementation provided is that there are three parameters that we may wish to tweak to improve the performance of our model. These are the embedding dimension, the hidden dimension and the size of the vocabulary. We will likely want to make these parameters configurable in the training script so that if we wish to modify them we do not need to modify the script itself. We will see how to do this later on. To start we will write some of the training code in the notebook so that we can more easily diagnose any issues that arise.

First we will load a small portion of the training data set to use as a sample. It would be very time consuming to try and train the model completely in the notebook as we do not have access to a gpu and the compute instance that we are using is not particularly powerful. However, we can work on a small bit of the data to get a feel for how our training script is behaving.


```python
import torch
import torch.utils.data

# Read in only the first 250 rows
train_sample = pd.read_csv(os.path.join(data_dir, 'train.csv'), header=None, names=None, nrows=250)

# Turn the input pandas dataframe into tensors
train_sample_y = torch.from_numpy(train_sample[[0]].values).float().squeeze()
train_sample_X = torch.from_numpy(train_sample.drop([0], axis=1).values).long()

# Build the dataset
train_sample_ds = torch.utils.data.TensorDataset(train_sample_X, train_sample_y)
# Build the dataloader
train_sample_dl = torch.utils.data.DataLoader(train_sample_ds, batch_size=50)
```

### (TODO) Writing the training method

Next we need to write the training code itself. This should be very similar to training methods that you have written before to train PyTorch models. We will leave any difficult aspects such as model saving / loading and parameter loading until a little later.


```python
def train(model, train_loader, epochs, optimizer, loss_fn, device):
    for epoch in range(1, epochs + 1):
        model.train()
        total_loss = 0
        for batch in train_loader:         
            batch_X, batch_y = batch
            
            batch_X = batch_X.to(device)
            batch_y = batch_y.to(device)
            
            # TODO: Complete this train method to train the model provided.
            optimizer.zero_grad()
            out = model.forward(batch_X)
            loss = loss_fn(out, batch_y)
            loss.backward()
            optimizer.step()
            
            total_loss += loss.data.item()
        print("Epoch: {}, BCELoss: {}".format(epoch, total_loss / len(train_loader)))
```

Supposing we have the training method above, we will test that it is working by writing a bit of code in the notebook that executes our training method on the small sample training set that we loaded earlier. The reason for doing this in the notebook is so that we have an opportunity to fix any errors that arise early when they are easier to diagnose.


```python
import torch.optim as optim
from train.model import LSTMClassifier

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = LSTMClassifier(32, 100, 5000).to(device)
optimizer = optim.Adam(model.parameters())
loss_fn = torch.nn.BCELoss()

train(model, train_sample_dl, 5, optimizer, loss_fn, device)
```

    Epoch: 1, BCELoss: 0.6949911832809448
    Epoch: 2, BCELoss: 0.6822074055671692
    Epoch: 3, BCELoss: 0.6717594981193542
    Epoch: 4, BCELoss: 0.6617854237556458
    Epoch: 5, BCELoss: 0.6516794562339783


In order to construct a PyTorch model using SageMaker we must provide SageMaker with a training script. We may optionally include a directory which will be copied to the container and from which our training code will be run. When the training container is executed it will check the uploaded directory (if there is one) for a `requirements.txt` file and install any required Python libraries, after which the training script will be run.

### (TODO) Training the model

When a PyTorch model is constructed in SageMaker, an entry point must be specified. This is the Python file which will be executed when the model is trained. Inside of the `train` directory is a file called `train.py` which has been provided and which contains most of the necessary code to train our model. The only thing that is missing is the implementation of the `train()` method which you wrote earlier in this notebook.

**TODO**: Copy the `train()` method written above and paste it into the `train/train.py` file where required.

The way that SageMaker passes hyperparameters to the training script is by way of arguments. These arguments can then be parsed and used in the training script. To see how this is done take a look at the provided `train/train.py` file.


```python
from sagemaker.pytorch import PyTorch

estimator = PyTorch(entry_point="train.py",
                    source_dir="train",
                    role=role,
                    framework_version='0.4.0',
                    train_instance_count=1,
                    train_instance_type='ml.p2.xlarge',
                    hyperparameters={
                        'epochs': 10,
                        'hidden_dim': 200,
                    })
```


```python
estimator.fit({'training': input_data})
```

    'create_image_uri' will be deprecated in favor of 'ImageURIProvider' class in SageMaker Python SDK v2.
    's3_input' class will be renamed to 'TrainingInput' in SageMaker Python SDK v2.
    'create_image_uri' will be deprecated in favor of 'ImageURIProvider' class in SageMaker Python SDK v2.


    2020-09-10 18:31:58 Starting - Starting the training job...
    2020-09-10 18:32:00 Starting - Launching requested ML instances......
    2020-09-10 18:33:23 Starting - Preparing the instances for training............
    2020-09-10 18:35:11 Downloading - Downloading input data...
    2020-09-10 18:35:40 Training - Downloading the training image..[34mbash: cannot set terminal process group (-1): Inappropriate ioctl for device[0m
    [34mbash: no job control in this shell[0m
    [34m2020-09-10 18:36:02,046 sagemaker-containers INFO     Imported framework sagemaker_pytorch_container.training[0m
    [34m2020-09-10 18:36:02,071 sagemaker_pytorch_container.training INFO     Block until all host DNS lookups succeed.[0m
    [34m2020-09-10 18:36:02,075 sagemaker_pytorch_container.training INFO     Invoking user training script.[0m
    [34m2020-09-10 18:36:02,337 sagemaker-containers INFO     Module train does not provide a setup.py. [0m
    [34mGenerating setup.py[0m
    [34m2020-09-10 18:36:02,338 sagemaker-containers INFO     Generating setup.cfg[0m
    [34m2020-09-10 18:36:02,338 sagemaker-containers INFO     Generating MANIFEST.in[0m
    [34m2020-09-10 18:36:02,338 sagemaker-containers INFO     Installing module with the following command:[0m
    [34m/usr/bin/python -m pip install -U . -r requirements.txt[0m
    [34mProcessing /opt/ml/code[0m
    [34mCollecting pandas (from -r requirements.txt (line 1))
      Downloading https://files.pythonhosted.org/packages/74/24/0cdbf8907e1e3bc5a8da03345c23cbed7044330bb8f73bb12e711a640a00/pandas-0.24.2-cp35-cp35m-manylinux1_x86_64.whl (10.0MB)[0m
    [34mCollecting numpy (from -r requirements.txt (line 2))[0m
    [34m  Downloading https://files.pythonhosted.org/packages/b5/36/88723426b4ff576809fec7d73594fe17a35c27f8d01f93637637a29ae25b/numpy-1.18.5-cp35-cp35m-manylinux1_x86_64.whl (19.9MB)[0m
    [34mCollecting nltk (from -r requirements.txt (line 3))
      Downloading https://files.pythonhosted.org/packages/92/75/ce35194d8e3022203cca0d2f896dbb88689f9b3fce8e9f9cff942913519d/nltk-3.5.zip (1.4MB)[0m
    [34mCollecting beautifulsoup4 (from -r requirements.txt (line 4))
      Downloading https://files.pythonhosted.org/packages/66/25/ff030e2437265616a1e9b25ccc864e0371a0bc3adb7c5a404fd661c6f4f6/beautifulsoup4-4.9.1-py3-none-any.whl (115kB)[0m
    [34mCollecting html5lib (from -r requirements.txt (line 5))
      Downloading https://files.pythonhosted.org/packages/6c/dd/a834df6482147d48e225a49515aabc28974ad5a4ca3215c18a882565b028/html5lib-1.1-py2.py3-none-any.whl (112kB)[0m
    [34mCollecting pytz>=2011k (from pandas->-r requirements.txt (line 1))
      Downloading https://files.pythonhosted.org/packages/4f/a4/879454d49688e2fad93e59d7d4efda580b783c745fd2ec2a3adf87b0808d/pytz-2020.1-py2.py3-none-any.whl (510kB)[0m
    [34mRequirement already satisfied, skipping upgrade: python-dateutil>=2.5.0 in /usr/local/lib/python3.5/dist-packages (from pandas->-r requirements.txt (line 1)) (2.7.5)[0m
    [34mRequirement already satisfied, skipping upgrade: click in /usr/local/lib/python3.5/dist-packages (from nltk->-r requirements.txt (line 3)) (7.0)[0m
    [34mCollecting joblib (from nltk->-r requirements.txt (line 3))
      Downloading https://files.pythonhosted.org/packages/28/5c/cf6a2b65a321c4a209efcdf64c2689efae2cb62661f8f6f4bb28547cf1bf/joblib-0.14.1-py2.py3-none-any.whl (294kB)[0m
    [34mCollecting regex (from nltk->-r requirements.txt (line 3))
      Downloading https://files.pythonhosted.org/packages/09/c3/ddaa87500f31ed86290e3d014c0302a51fde28d7139eda0b5f33733726db/regex-2020.7.14.tar.gz (690kB)[0m
    [34mCollecting tqdm (from nltk->-r requirements.txt (line 3))
      Downloading https://files.pythonhosted.org/packages/28/7e/281edb5bc3274dfb894d90f4dbacfceaca381c2435ec6187a2c6f329aed7/tqdm-4.48.2-py2.py3-none-any.whl (68kB)[0m
    [34mCollecting soupsieve>1.2 (from beautifulsoup4->-r requirements.txt (line 4))
      Downloading https://files.pythonhosted.org/packages/6f/8f/457f4a5390eeae1cc3aeab89deb7724c965be841ffca6cfca9197482e470/soupsieve-2.0.1-py3-none-any.whl[0m
    [34mRequirement already satisfied, skipping upgrade: six>=1.9 in /usr/local/lib/python3.5/dist-packages (from html5lib->-r requirements.txt (line 5)) (1.11.0)[0m
    [34mCollecting webencodings (from html5lib->-r requirements.txt (line 5))
      Downloading https://files.pythonhosted.org/packages/f4/24/2a3e3df732393fed8b3ebf2ec078f05546de641fe1b667ee316ec1dcf3b7/webencodings-0.5.1-py2.py3-none-any.whl[0m
    [34mBuilding wheels for collected packages: nltk, train, regex
      Running setup.py bdist_wheel for nltk: started[0m
    [34m  Running setup.py bdist_wheel for nltk: finished with status 'done'
      Stored in directory: /root/.cache/pip/wheels/ae/8c/3f/b1fe0ba04555b08b57ab52ab7f86023639a526d8bc8d384306
      Running setup.py bdist_wheel for train: started
      Running setup.py bdist_wheel for train: finished with status 'done'
      Stored in directory: /tmp/pip-ephem-wheel-cache-o34sj5df/wheels/35/24/16/37574d11bf9bde50616c67372a334f94fa8356bc7164af8ca3
      Running setup.py bdist_wheel for regex: started[0m
    
    2020-09-10 18:36:01 Training - Training image download completed. Training in progress.[34m  Running setup.py bdist_wheel for regex: finished with status 'done'
      Stored in directory: /root/.cache/pip/wheels/53/55/dc/e17fa4568958f4c53be34b65e474a1327b64641f65df379ec3[0m
    [34mSuccessfully built nltk train regex[0m
    [34mInstalling collected packages: numpy, pytz, pandas, joblib, regex, tqdm, nltk, soupsieve, beautifulsoup4, webencodings, html5lib, train
      Found existing installation: numpy 1.15.4
        Uninstalling numpy-1.15.4:
          Successfully uninstalled numpy-1.15.4[0m
    [34mSuccessfully installed beautifulsoup4-4.9.1 html5lib-1.1 joblib-0.14.1 nltk-3.5 numpy-1.18.5 pandas-0.24.2 pytz-2020.1 regex-2020.7.14 soupsieve-2.0.1 tqdm-4.48.2 train-1.0.0 webencodings-0.5.1[0m
    [34mYou are using pip version 18.1, however version 20.2.3 is available.[0m
    [34mYou should consider upgrading via the 'pip install --upgrade pip' command.[0m
    [34m2020-09-10 18:36:24,591 sagemaker-containers INFO     Invoking user script
    [0m
    [34mTraining Env:
    [0m
    [34m{
        "module_name": "train",
        "job_name": "sagemaker-pytorch-2020-09-10-18-31-58-277",
        "num_cpus": 4,
        "hosts": [
            "algo-1"
        ],
        "channel_input_dirs": {
            "training": "/opt/ml/input/data/training"
        },
        "network_interface_name": "eth0",
        "output_dir": "/opt/ml/output",
        "current_host": "algo-1",
        "input_config_dir": "/opt/ml/input/config",
        "user_entry_point": "train.py",
        "framework_module": "sagemaker_pytorch_container.training:main",
        "resource_config": {
            "network_interface_name": "eth0",
            "hosts": [
                "algo-1"
            ],
            "current_host": "algo-1"
        },
        "log_level": 20,
        "hyperparameters": {
            "epochs": 10,
            "hidden_dim": 200
        },
        "output_intermediate_dir": "/opt/ml/output/intermediate",
        "additional_framework_parameters": {},
        "input_data_config": {
            "training": {
                "S3DistributionType": "FullyReplicated",
                "RecordWrapperType": "None",
                "TrainingInputMode": "File"
            }
        },
        "model_dir": "/opt/ml/model",
        "input_dir": "/opt/ml/input",
        "num_gpus": 1,
        "output_data_dir": "/opt/ml/output/data",
        "module_dir": "s3://sagemaker-us-east-2-423904393887/sagemaker-pytorch-2020-09-10-18-31-58-277/source/sourcedir.tar.gz"[0m
    [34m}
    [0m
    [34mEnvironment variables:
    [0m
    [34mSM_LOG_LEVEL=20[0m
    [34mSM_USER_ARGS=["--epochs","10","--hidden_dim","200"][0m
    [34mSM_INPUT_DIR=/opt/ml/input[0m
    [34mSM_HOSTS=["algo-1"][0m
    [34mSM_HP_HIDDEN_DIM=200[0m
    [34mSM_MODULE_DIR=s3://sagemaker-us-east-2-423904393887/sagemaker-pytorch-2020-09-10-18-31-58-277/source/sourcedir.tar.gz[0m
    [34mSM_FRAMEWORK_MODULE=sagemaker_pytorch_container.training:main[0m
    [34mSM_FRAMEWORK_PARAMS={}[0m
    [34mSM_NUM_GPUS=1[0m
    [34mSM_CHANNEL_TRAINING=/opt/ml/input/data/training[0m
    [34mSM_OUTPUT_DATA_DIR=/opt/ml/output/data[0m
    [34mSM_OUTPUT_DIR=/opt/ml/output[0m
    [34mSM_NUM_CPUS=4[0m
    [34mSM_MODULE_NAME=train[0m
    [34mSM_INPUT_DATA_CONFIG={"training":{"RecordWrapperType":"None","S3DistributionType":"FullyReplicated","TrainingInputMode":"File"}}[0m
    [34mSM_MODEL_DIR=/opt/ml/model[0m
    [34mSM_CHANNELS=["training"][0m
    [34mSM_NETWORK_INTERFACE_NAME=eth0[0m
    [34mPYTHONPATH=/usr/local/bin:/usr/lib/python35.zip:/usr/lib/python3.5:/usr/lib/python3.5/plat-x86_64-linux-gnu:/usr/lib/python3.5/lib-dynload:/usr/local/lib/python3.5/dist-packages:/usr/lib/python3/dist-packages[0m
    [34mSM_HP_EPOCHS=10[0m
    [34mSM_USER_ENTRY_POINT=train.py[0m
    [34mSM_TRAINING_ENV={"additional_framework_parameters":{},"channel_input_dirs":{"training":"/opt/ml/input/data/training"},"current_host":"algo-1","framework_module":"sagemaker_pytorch_container.training:main","hosts":["algo-1"],"hyperparameters":{"epochs":10,"hidden_dim":200},"input_config_dir":"/opt/ml/input/config","input_data_config":{"training":{"RecordWrapperType":"None","S3DistributionType":"FullyReplicated","TrainingInputMode":"File"}},"input_dir":"/opt/ml/input","job_name":"sagemaker-pytorch-2020-09-10-18-31-58-277","log_level":20,"model_dir":"/opt/ml/model","module_dir":"s3://sagemaker-us-east-2-423904393887/sagemaker-pytorch-2020-09-10-18-31-58-277/source/sourcedir.tar.gz","module_name":"train","network_interface_name":"eth0","num_cpus":4,"num_gpus":1,"output_data_dir":"/opt/ml/output/data","output_dir":"/opt/ml/output","output_intermediate_dir":"/opt/ml/output/intermediate","resource_config":{"current_host":"algo-1","hosts":["algo-1"],"network_interface_name":"eth0"},"user_entry_point":"train.py"}[0m
    [34mSM_OUTPUT_INTERMEDIATE_DIR=/opt/ml/output/intermediate[0m
    [34mSM_CURRENT_HOST=algo-1[0m
    [34mSM_HPS={"epochs":10,"hidden_dim":200}[0m
    [34mSM_RESOURCE_CONFIG={"current_host":"algo-1","hosts":["algo-1"],"network_interface_name":"eth0"}[0m
    [34mSM_INPUT_CONFIG_DIR=/opt/ml/input/config
    [0m
    [34mInvoking script with the following command:
    [0m
    [34m/usr/bin/python -m train --epochs 10 --hidden_dim 200
    
    [0m
    [34mUsing device cuda.[0m
    [34mGet train data loader.[0m
    [34mModel loaded with embedding_dim 32, hidden_dim 200, vocab_size 5000.[0m
    [34m2020-09-10 18:36:33,537 sagemaker-containers INFO     Reporting training SUCCESS[0m
    
    2020-09-10 18:36:43 Uploading - Uploading generated training model
    2020-09-10 18:36:43 Completed - Training job completed
    Training seconds: 92
    Billable seconds: 92


## Step 5: Testing the model

As mentioned at the top of this notebook, we will be testing this model by first deploying it and then sending the testing data to the deployed endpoint. We will do this so that we can make sure that the deployed model is working correctly.

## Step 6: Deploy the model for testing

Now that we have trained our model, we would like to test it to see how it performs. Currently our model takes input of the form `review_length, review[500]` where `review[500]` is a sequence of `500` integers which describe the words present in the review, encoded using `word_dict`. Fortunately for us, SageMaker provides built-in inference code for models with simple inputs such as this.

There is one thing that we need to provide, however, and that is a function which loads the saved model. This function must be called `model_fn()` and takes as its only parameter a path to the directory where the model artifacts are stored. This function must also be present in the python file which we specified as the entry point. In our case the model loading function has been provided and so no changes need to be made.

**NOTE**: When the built-in inference code is run it must import the `model_fn()` method from the `train.py` file. This is why the training code is wrapped in a main guard ( ie, `if __name__ == '__main__':` )

Since we don't need to change anything in the code that was uploaded during training, we can simply deploy the current model as-is.

**NOTE:** When deploying a model you are asking SageMaker to launch an compute instance that will wait for data to be sent to it. As a result, this compute instance will continue to run until *you* shut it down. This is important to know since the cost of a deployed endpoint depends on how long it has been running for.

In other words **If you are no longer using a deployed endpoint, shut it down!**

**TODO:** Deploy the trained model.


```python
# TODO: Deploy the trained model
predictor = estimator.deploy(initial_instance_count=1, instance_type='ml.p2.xlarge')
```

    Parameter image will be renamed to image_uri in SageMaker Python SDK v2.
    'create_image_uri' will be deprecated in favor of 'ImageURIProvider' class in SageMaker Python SDK v2.


    -------------------!

## Step 7 - Use the model for testing

Once deployed, we can read in the test data and send it off to our deployed model to get some results. Once we collect all of the results we can determine how accurate our model is.


```python
test_X = pd.concat([pd.DataFrame(test_X_len), pd.DataFrame(test_X)], axis=1)
```


```python
# We split the data into chunks and send each chunk seperately, accumulating the results.

def predict(data, rows=512):
    split_array = np.array_split(data, int(data.shape[0] / float(rows) + 1))
    predictions = np.array([])
    for array in split_array:
        predictions = np.append(predictions, predictor.predict(array))
    
    return predictions
```


```python
predictions = predict(test_X.values)
predictions = [round(num) for num in predictions]
```


```python
from sklearn.metrics import accuracy_score
accuracy_score(test_y, predictions)
```




    0.50548



**Question:** How does this model compare to the XGBoost model you created earlier? Why might these two models perform differently on this dataset? Which do *you* think is better for sentiment analysis?

**Answer:**
The previously created XGBoost model performed a little better than the current model. One key difference to account for is that XGBoosting is primarily a tree based/gradient boosting model that uses a matrix enhanced feature table(consisting of counts or tf-idf) which may be independent from the order of words in the data. The XGBoosting model would be more accurate in terms of accuracy scores and predictions, but may not be the best choice is the objective is to have a model that is highly context drive, making the former more appropriate for smaller datasets that have a single label approach to them. The Pytorch model may be improved with advanced hyper-parameter tuning but for now XGBoost seems like a better approach for sentiment analysis.

### (TODO) More testing

We now have a trained model which has been deployed and which we can send processed reviews to and which returns the predicted sentiment. However, ultimately we would like to be able to send our model an unprocessed review. That is, we would like to send the review itself as a string. For example, suppose we wish to send the following review to our model.


```python
test_review = 'The simplest pleasures in life are the best, and this film is one of them. Combining a rather basic storyline of love and adventure this movie transcends the usual weekend fair with wit and unmitigated charm.'
```

The question we now need to answer is, how do we send this review to our model?

Recall in the first section of this notebook we did a bunch of data processing to the IMDb dataset. In particular, we did two specific things to the provided reviews.
 - Removed any html tags and stemmed the input
 - Encoded the review as a sequence of integers using `word_dict`
 
In order process the review we will need to repeat these two steps.

**TODO**: Using the `review_to_words` and `convert_and_pad` methods from section one, convert `test_review` into a numpy array `test_data` suitable to send to our model. Remember that our model expects input of the form `review_length, review[500]`.


```python
# TODO: Convert test_review into a form usable by the model and save the results in test_data
test_data = review_to_words(test_review)
test_data = [np.array(convert_and_pad(word_dict, test_data)[0])]
```


```python
test_review_words = review_to_words(test_review)     # splits reviews to words
review_X, review_len = convert_and_pad(word_dict, test_review_words)   # pad review

data_pack = np.hstack((review_len, review_X))
data_pack = data_pack.reshape(1, -1)

test_data = torch.from_numpy(data_pack)
test_data = test_data.to(device)
```

Now that we have processed the review, we can send the resulting array to our model to predict the sentiment of the review.


```python
predictor.predict(test_data)
```




    array(0.5066693, dtype=float32)



Since the return value of our model is close to `1`, we can be certain that the review we submitted is positive.

### Delete the endpoint

Of course, just like in the XGBoost notebook, once we've deployed an endpoint it continues to run until we tell it to shut down. Since we are done using our endpoint for now, we can delete it.


```python
estimator.delete_endpoint()
```

    estimator.delete_endpoint() will be deprecated in SageMaker Python SDK v2. Please use the delete_endpoint() function on your predictor instead.


## Step 6 (again) - Deploy the model for the web app

Now that we know that our model is working, it's time to create some custom inference code so that we can send the model a review which has not been processed and have it determine the sentiment of the review.

As we saw above, by default the estimator which we created, when deployed, will use the entry script and directory which we provided when creating the model. However, since we now wish to accept a string as input and our model expects a processed review, we need to write some custom inference code.

We will store the code that we write in the `serve` directory. Provided in this directory is the `model.py` file that we used to construct our model, a `utils.py` file which contains the `review_to_words` and `convert_and_pad` pre-processing functions which we used during the initial data processing, and `predict.py`, the file which will contain our custom inference code. Note also that `requirements.txt` is present which will tell SageMaker what Python libraries are required by our custom inference code.

When deploying a PyTorch model in SageMaker, you are expected to provide four functions which the SageMaker inference container will use.
 - `model_fn`: This function is the same function that we used in the training script and it tells SageMaker how to load our model.
 - `input_fn`: This function receives the raw serialized input that has been sent to the model's endpoint and its job is to de-serialize and make the input available for the inference code.
 - `output_fn`: This function takes the output of the inference code and its job is to serialize this output and return it to the caller of the model's endpoint.
 - `predict_fn`: The heart of the inference script, this is where the actual prediction is done and is the function which you will need to complete.

For the simple website that we are constructing during this project, the `input_fn` and `output_fn` methods are relatively straightforward. We only require being able to accept a string as input and we expect to return a single value as output. You might imagine though that in a more complex application the input or output may be image data or some other binary data which would require some effort to serialize.

### (TODO) Writing inference code

Before writing our custom inference code, we will begin by taking a look at the code which has been provided.


```python
!pygmentize serve/predict.py
```

    [34mimport[39;49;00m [04m[36margparse[39;49;00m
    [34mimport[39;49;00m [04m[36mjson[39;49;00m
    [34mimport[39;49;00m [04m[36mos[39;49;00m
    [34mimport[39;49;00m [04m[36mpickle[39;49;00m
    [34mimport[39;49;00m [04m[36msys[39;49;00m
    [34mimport[39;49;00m [04m[36msagemaker_containers[39;49;00m
    [34mimport[39;49;00m [04m[36mpandas[39;49;00m [34mas[39;49;00m [04m[36mpd[39;49;00m
    [34mimport[39;49;00m [04m[36mnumpy[39;49;00m [34mas[39;49;00m [04m[36mnp[39;49;00m
    [34mimport[39;49;00m [04m[36mtorch[39;49;00m
    [34mimport[39;49;00m [04m[36mtorch[39;49;00m[04m[36m.[39;49;00m[04m[36mnn[39;49;00m [34mas[39;49;00m [04m[36mnn[39;49;00m
    [34mimport[39;49;00m [04m[36mtorch[39;49;00m[04m[36m.[39;49;00m[04m[36moptim[39;49;00m [34mas[39;49;00m [04m[36moptim[39;49;00m
    [34mimport[39;49;00m [04m[36mtorch[39;49;00m[04m[36m.[39;49;00m[04m[36mutils[39;49;00m[04m[36m.[39;49;00m[04m[36mdata[39;49;00m
    
    [34mfrom[39;49;00m [04m[36mmodel[39;49;00m [34mimport[39;49;00m LSTMClassifier
    
    [34mfrom[39;49;00m [04m[36mutils[39;49;00m [34mimport[39;49;00m review_to_words, convert_and_pad
    
    [34mdef[39;49;00m [32mmodel_fn[39;49;00m(model_dir):
        [33m"""Load the PyTorch model from the `model_dir` directory."""[39;49;00m
        [36mprint[39;49;00m([33m"[39;49;00m[33mLoading model.[39;49;00m[33m"[39;49;00m)
    
        [37m# First, load the parameters used to create the model.[39;49;00m
        model_info = {}
        model_info_path = os.path.join(model_dir, [33m'[39;49;00m[33mmodel_info.pth[39;49;00m[33m'[39;49;00m)
        [34mwith[39;49;00m [36mopen[39;49;00m(model_info_path, [33m'[39;49;00m[33mrb[39;49;00m[33m'[39;49;00m) [34mas[39;49;00m f:
            model_info = torch.load(f)
    
        [36mprint[39;49;00m([33m"[39;49;00m[33mmodel_info: [39;49;00m[33m{}[39;49;00m[33m"[39;49;00m.format(model_info))
    
        [37m# Determine the device and construct the model.[39;49;00m
        device = torch.device([33m"[39;49;00m[33mcuda[39;49;00m[33m"[39;49;00m [34mif[39;49;00m torch.cuda.is_available() [34melse[39;49;00m [33m"[39;49;00m[33mcpu[39;49;00m[33m"[39;49;00m)
        model = LSTMClassifier(model_info[[33m'[39;49;00m[33membedding_dim[39;49;00m[33m'[39;49;00m], model_info[[33m'[39;49;00m[33mhidden_dim[39;49;00m[33m'[39;49;00m], model_info[[33m'[39;49;00m[33mvocab_size[39;49;00m[33m'[39;49;00m])
    
        [37m# Load the store model parameters.[39;49;00m
        model_path = os.path.join(model_dir, [33m'[39;49;00m[33mmodel.pth[39;49;00m[33m'[39;49;00m)
        [34mwith[39;49;00m [36mopen[39;49;00m(model_path, [33m'[39;49;00m[33mrb[39;49;00m[33m'[39;49;00m) [34mas[39;49;00m f:
            model.load_state_dict(torch.load(f))
    
        [37m# Load the saved word_dict.[39;49;00m
        word_dict_path = os.path.join(model_dir, [33m'[39;49;00m[33mword_dict.pkl[39;49;00m[33m'[39;49;00m)
        [34mwith[39;49;00m [36mopen[39;49;00m(word_dict_path, [33m'[39;49;00m[33mrb[39;49;00m[33m'[39;49;00m) [34mas[39;49;00m f:
            model.word_dict = pickle.load(f)
    
        model.to(device).eval()
    
        [36mprint[39;49;00m([33m"[39;49;00m[33mDone loading model.[39;49;00m[33m"[39;49;00m)
        [34mreturn[39;49;00m model
    
    [34mdef[39;49;00m [32minput_fn[39;49;00m(serialized_input_data, content_type):
        [36mprint[39;49;00m([33m'[39;49;00m[33mDeserializing the input data.[39;49;00m[33m'[39;49;00m)
        [34mif[39;49;00m content_type == [33m'[39;49;00m[33mtext/plain[39;49;00m[33m'[39;49;00m:
            data = serialized_input_data.decode([33m'[39;49;00m[33mutf-8[39;49;00m[33m'[39;49;00m)
            [34mreturn[39;49;00m data
        [34mraise[39;49;00m [36mException[39;49;00m([33m'[39;49;00m[33mRequested unsupported ContentType in content_type: [39;49;00m[33m'[39;49;00m + content_type)
    
    [34mdef[39;49;00m [32moutput_fn[39;49;00m(prediction_output, accept):
        [36mprint[39;49;00m([33m'[39;49;00m[33mSerializing the generated output.[39;49;00m[33m'[39;49;00m)
        [34mreturn[39;49;00m [36mstr[39;49;00m(prediction_output)
    
    [34mdef[39;49;00m [32mpredict_fn[39;49;00m(input_data, model):
        [36mprint[39;49;00m([33m'[39;49;00m[33mInferring sentiment of input data.[39;49;00m[33m'[39;49;00m)
    
        device = torch.device([33m"[39;49;00m[33mcuda[39;49;00m[33m"[39;49;00m [34mif[39;49;00m torch.cuda.is_available() [34melse[39;49;00m [33m"[39;49;00m[33mcpu[39;49;00m[33m"[39;49;00m)
        
        [34mif[39;49;00m model.word_dict [35mis[39;49;00m [34mNone[39;49;00m:
            [34mraise[39;49;00m [36mException[39;49;00m([33m'[39;49;00m[33mModel has not been loaded properly, no word_dict.[39;49;00m[33m'[39;49;00m)
        
        [37m# TODO: Process input_data so that it is ready to be sent to our model.[39;49;00m
        [37m#       You should produce two variables:[39;49;00m
        [37m#         data_X   - A sequence of length 500 which represents the converted review[39;49;00m
        [37m#         data_len - The length of the review[39;49;00m
    
        data_X = [34mNone[39;49;00m
        data_len = [34mNone[39;49;00m
    
        [37m# Using data_X and data_len we construct an appropriate input tensor. Remember[39;49;00m
        [37m# that our model expects input data of the form 'len, review[500]'.[39;49;00m
        data_pack = np.hstack((data_len, data_X))
        data_pack = data_pack.reshape([34m1[39;49;00m, -[34m1[39;49;00m)
        
        data = torch.from_numpy(data_pack)
        data = data.to(device)
    
        [37m# Make sure to put the model into evaluation mode[39;49;00m
        model.eval()
    
        [37m# TODO: Compute the result of applying the model to the input data. The variable `result` should[39;49;00m
        [37m#       be a numpy array which contains a single integer which is either 1 or 0[39;49;00m
    
        result = [34mNone[39;49;00m
    
        [34mreturn[39;49;00m result


As mentioned earlier, the `model_fn` method is the same as the one provided in the training code and the `input_fn` and `output_fn` methods are very simple and your task will be to complete the `predict_fn` method. Make sure that you save the completed file as `predict.py` in the `serve` directory.

**TODO**: Complete the `predict_fn()` method in the `serve/predict.py` file.

### Deploying the model

Now that the custom inference code has been written, we will create and deploy our model. To begin with, we need to construct a new PyTorchModel object which points to the model artifacts created during training and also points to the inference code that we wish to use. Then we can call the deploy method to launch the deployment container.

**NOTE**: The default behaviour for a deployed PyTorch model is to assume that any input passed to the predictor is a `numpy` array. In our case we want to send a string so we need to construct a simple wrapper around the `RealTimePredictor` class to accomodate simple strings. In a more complicated situation you may want to provide a serialization object, for example if you wanted to sent image data.


```python
from sagemaker.predictor import RealTimePredictor
from sagemaker.pytorch import PyTorchModel

class StringPredictor(RealTimePredictor):
    def __init__(self, endpoint_name, sagemaker_session):
        super(StringPredictor, self).__init__(endpoint_name, sagemaker_session, content_type='text/plain')

model = PyTorchModel(model_data=estimator.model_data,
                     role = role,
                     framework_version='0.4.0',
                     entry_point='predict.py',
                     source_dir='serve',
                     predictor_cls=StringPredictor)
predictor = model.deploy(initial_instance_count=1, instance_type='ml.m4.xlarge')
```

    Parameter image will be renamed to image_uri in SageMaker Python SDK v2.
    'create_image_uri' will be deprecated in favor of 'ImageURIProvider' class in SageMaker Python SDK v2.


    ---------------!

### Testing the model

Now that we have deployed our model with the custom inference code, we should test to see if everything is working. Here we test our model by loading the first `250` positive and negative reviews and send them to the endpoint, then collect the results. The reason for only sending some of the data is that the amount of time it takes for our model to process the input and then perform inference is quite long and so testing the entire data set would be prohibitive.


```python
import glob

def test_reviews(data_dir='../data/aclImdb', stop=250):
    
    results = []
    ground = []
    
    # We make sure to test both positive and negative reviews    
    for sentiment in ['pos', 'neg']:
        
        path = os.path.join(data_dir, 'test', sentiment, '*.txt')
        files = glob.glob(path)
        
        files_read = 0
        
        print('Starting ', sentiment, ' files')
        
        # Iterate through the files and send them to the predictor
        for f in files:
            with open(f) as review:
                # First, we store the ground truth (was the review positive or negative)
                if sentiment == 'pos':
                    ground.append(1)
                else:
                    ground.append(0)
                # Read in the review and convert to 'utf-8' for transmission via HTTP
                review_input = review.read().encode('utf-8')
                # Send the review to the predictor and store the results
                results.append(int(predictor.predict(review_input)))
                
            # Sending reviews to our endpoint one at a time takes a while so we
            # only send a small number of reviews
            files_read += 1
            if files_read == stop:
                break
            
    return ground, results
```


```python
ground, results = test_reviews()
```

    Starting  pos  files



    ---------------------------------------------------------------------------

    ValidationError                           Traceback (most recent call last)

    <ipython-input-52-27d1fd4b7c7b> in <module>
    ----> 1 ground, results = test_reviews()
    

    <ipython-input-43-4c56d4ff716b> in test_reviews(data_dir, stop)
         27                 review_input = review.read().encode('utf-8')
         28                 # Send the review to the predictor and store the results
    ---> 29                 results.append(int(predictor.predict(review_input)))
         30 
         31             # Sending reviews to our endpoint one at a time takes a while so we


    ~/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages/sagemaker/predictor.py in predict(self, data, initial_args, target_model, target_variant)
        111 
        112         request_args = self._create_request_args(data, initial_args, target_model, target_variant)
    --> 113         response = self.sagemaker_session.sagemaker_runtime_client.invoke_endpoint(**request_args)
        114         return self._handle_response(response)
        115 


    ~/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages/botocore/client.py in _api_call(self, *args, **kwargs)
        335                     "%s() only accepts keyword arguments." % py_operation_name)
        336             # The "self" in this scope is referring to the BaseClient.
    --> 337             return self._make_api_call(operation_name, kwargs)
        338 
        339         _api_call.__name__ = str(py_operation_name)


    ~/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages/botocore/client.py in _make_api_call(self, operation_name, api_params)
        654             error_code = parsed_response.get("Error", {}).get("Code")
        655             error_class = self.exceptions.from_code(error_code)
    --> 656             raise error_class(parsed_response, operation_name)
        657         else:
        658             return parsed_response


    ValidationError: An error occurred (ValidationError) when calling the InvokeEndpoint operation: Endpoint sagemaker-pytorch-2020-09-10-08-46-11-995 of account 423904393887 not found.



```python
from sklearn.metrics import accuracy_score
accuracy_score(ground, results)
```


    ---------------------------------------------------------------------------

    NameError                                 Traceback (most recent call last)

    <ipython-input-42-f3e6875633e1> in <module>
          1 from sklearn.metrics import accuracy_score
    ----> 2 accuracy_score(ground, results)
    

    NameError: name 'ground' is not defined


As an additional test, we can try sending the `test_review` that we looked at earlier.


```python
predictor.predict(test_review)
```


    ---------------------------------------------------------------------------

    ModelError                                Traceback (most recent call last)

    <ipython-input-47-ccef8a3a2cb2> in <module>
    ----> 1 predictor.predict(test_review)
    

    ~/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages/sagemaker/predictor.py in predict(self, data, initial_args, target_model, target_variant)
        111 
        112         request_args = self._create_request_args(data, initial_args, target_model, target_variant)
    --> 113         response = self.sagemaker_session.sagemaker_runtime_client.invoke_endpoint(**request_args)
        114         return self._handle_response(response)
        115 


    ~/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages/botocore/client.py in _api_call(self, *args, **kwargs)
        335                     "%s() only accepts keyword arguments." % py_operation_name)
        336             # The "self" in this scope is referring to the BaseClient.
    --> 337             return self._make_api_call(operation_name, kwargs)
        338 
        339         _api_call.__name__ = str(py_operation_name)


    ~/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages/botocore/client.py in _make_api_call(self, operation_name, api_params)
        654             error_code = parsed_response.get("Error", {}).get("Code")
        655             error_class = self.exceptions.from_code(error_code)
    --> 656             raise error_class(parsed_response, operation_name)
        657         else:
        658             return parsed_response


    ModelError: An error occurred (ModelError) when calling the InvokeEndpoint operation: Received server error (500) from model with message "<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
    <title>500 Internal Server Error</title>
    <h1>Internal Server Error</h1>
    <p>The server encountered an internal error and was unable to complete your request.  Either the server is overloaded or there is an error in the application.</p>
    ". See https://us-east-2.console.aws.amazon.com/cloudwatch/home?region=us-east-2#logEventViewer:group=/aws/sagemaker/Endpoints/sagemaker-pytorch-2020-09-10-08-46-11-995 in account 423904393887 for more information.


Now that we know our endpoint is working as expected, we can set up the web page that will interact with it. If you don't have time to finish the project now, make sure to skip down to the end of this notebook and shut down your endpoint. You can deploy it again when you come back.

## Step 7 (again): Use the model for the web app

> **TODO:** This entire section and the next contain tasks for you to complete, mostly using the AWS console.

So far we have been accessing our model endpoint by constructing a predictor object which uses the endpoint and then just using the predictor object to perform inference. What if we wanted to create a web app which accessed our model? The way things are set up currently makes that not possible since in order to access a SageMaker endpoint the app would first have to authenticate with AWS using an IAM role which included access to SageMaker endpoints. However, there is an easier way! We just need to use some additional AWS services.

<img src="Web App Diagram.svg">

The diagram above gives an overview of how the various services will work together. On the far right is the model which we trained above and which is deployed using SageMaker. On the far left is our web app that collects a user's movie review, sends it off and expects a positive or negative sentiment in return.

In the middle is where some of the magic happens. We will construct a Lambda function, which you can think of as a straightforward Python function that can be executed whenever a specified event occurs. We will give this function permission to send and recieve data from a SageMaker endpoint.

Lastly, the method we will use to execute the Lambda function is a new endpoint that we will create using API Gateway. This endpoint will be a url that listens for data to be sent to it. Once it gets some data it will pass that data on to the Lambda function and then return whatever the Lambda function returns. Essentially it will act as an interface that lets our web app communicate with the Lambda function.

### Setting up a Lambda function

The first thing we are going to do is set up a Lambda function. This Lambda function will be executed whenever our public API has data sent to it. When it is executed it will receive the data, perform any sort of processing that is required, send the data (the review) to the SageMaker endpoint we've created and then return the result.

#### Part A: Create an IAM Role for the Lambda function

Since we want the Lambda function to call a SageMaker endpoint, we need to make sure that it has permission to do so. To do this, we will construct a role that we can later give the Lambda function.

Using the AWS Console, navigate to the **IAM** page and click on **Roles**. Then, click on **Create role**. Make sure that the **AWS service** is the type of trusted entity selected and choose **Lambda** as the service that will use this role, then click **Next: Permissions**.

In the search box type `sagemaker` and select the check box next to the **AmazonSageMakerFullAccess** policy. Then, click on **Next: Review**.

Lastly, give this role a name. Make sure you use a name that you will remember later on, for example `LambdaSageMakerRole`. Then, click on **Create role**.

#### Part B: Create a Lambda function

Now it is time to actually create the Lambda function.

Using the AWS Console, navigate to the AWS Lambda page and click on **Create a function**. When you get to the next page, make sure that **Author from scratch** is selected. Now, name your Lambda function, using a name that you will remember later on, for example `sentiment_analysis_func`. Make sure that the **Python 3.6** runtime is selected and then choose the role that you created in the previous part. Then, click on **Create Function**.

On the next page you will see some information about the Lambda function you've just created. If you scroll down you should see an editor in which you can write the code that will be executed when your Lambda function is triggered. In our example, we will use the code below. 

```python
# We need to use the low-level library to interact with SageMaker since the SageMaker API
# is not available natively through Lambda.
import boto3

def lambda_handler(event, context):

    # The SageMaker runtime is what allows us to invoke the endpoint that we've created.
    runtime = boto3.Session().client('sagemaker-runtime')

    # Now we use the SageMaker runtime to invoke our endpoint, sending the review we were given
    response = runtime.invoke_endpoint(EndpointName = '**ENDPOINT NAME HERE**',    # The name of the endpoint we created
                                       ContentType = 'text/plain',                 # The data format that is expected
                                       Body = event['body'])                       # The actual review

    # The response is an HTTP response whose body contains the result of our inference
    result = response['Body'].read().decode('utf-8')

    return {
        'statusCode' : 200,
        'headers' : { 'Content-Type' : 'text/plain', 'Access-Control-Allow-Origin' : '*' },
        'body' : result
    }
```

Once you have copy and pasted the code above into the Lambda code editor, replace the `**ENDPOINT NAME HERE**` portion with the name of the endpoint that we deployed earlier. You can determine the name of the endpoint using the code cell below.


```python
predictor.endpoint
```




    'sagemaker-pytorch-2020-09-10-08-46-11-995'



Once you have added the endpoint name to the Lambda function, click on **Save**. Your Lambda function is now up and running. Next we need to create a way for our web app to execute the Lambda function.

### Setting up API Gateway

Now that our Lambda function is set up, it is time to create a new API using API Gateway that will trigger the Lambda function we have just created.

Using AWS Console, navigate to **Amazon API Gateway** and then click on **Get started**.

On the next page, make sure that **New API** is selected and give the new api a name, for example, `sentiment_analysis_api`. Then, click on **Create API**.

Now we have created an API, however it doesn't currently do anything. What we want it to do is to trigger the Lambda function that we created earlier.

Select the **Actions** dropdown menu and click **Create Method**. A new blank method will be created, select its dropdown menu and select **POST**, then click on the check mark beside it.

For the integration point, make sure that **Lambda Function** is selected and click on the **Use Lambda Proxy integration**. This option makes sure that the data that is sent to the API is then sent directly to the Lambda function with no processing. It also means that the return value must be a proper response object as it will also not be processed by API Gateway.

Type the name of the Lambda function you created earlier into the **Lambda Function** text entry box and then click on **Save**. Click on **OK** in the pop-up box that then appears, giving permission to API Gateway to invoke the Lambda function you created.

The last step in creating the API Gateway is to select the **Actions** dropdown and click on **Deploy API**. You will need to create a new Deployment stage and name it anything you like, for example `prod`.

You have now successfully set up a public API to access your SageMaker model. Make sure to copy or write down the URL provided to invoke your newly created public API as this will be needed in the next step. This URL can be found at the top of the page, highlighted in blue next to the text **Invoke URL**.

## Step 4: Deploying our web app

Now that we have a publicly available API, we can start using it in a web app. For our purposes, we have provided a simple static html file which can make use of the public api you created earlier.

In the `website` folder there should be a file called `index.html`. Download the file to your computer and open that file up in a text editor of your choice. There should be a line which contains **\*\*REPLACE WITH PUBLIC API URL\*\***. Replace this string with the url that you wrote down in the last step and then save the file.

Now, if you open `index.html` on your local computer, your browser will behave as a local web server and you can use the provided site to interact with your SageMaker model.

If you'd like to go further, you can host this html file anywhere you'd like, for example using github or hosting a static site on Amazon's S3. Once you have done this you can share the link with anyone you'd like and have them play with it too!

> **Important Note** In order for the web app to communicate with the SageMaker endpoint, the endpoint has to actually be deployed and running. This means that you are paying for it. Make sure that the endpoint is running when you want to use the web app but that you shut it down when you don't need it, otherwise you will end up with a surprisingly large AWS bill.

**TODO:** Make sure that you include the edited `index.html` file in your project submission.

Now that your web app is working, trying playing around with it and see how well it works.

**Question**: Give an example of a review that you entered into your web app. What was the predicted sentiment of your example review?

**Answer:** 
The movie was well paced with amazing dialogue and scenes. A definite must watch.
API RESULT:- Positive.

### Delete the endpoint

Remember to always shut down your endpoint if you are no longer using it. You are charged for the length of time that the endpoint is running so if you forget and leave it on you could end up with an unexpectedly large bill.


```python
predictor.delete_endpoint()
```


```python
!pip install jupyter-cjk-xelatex
```

    Collecting jupyter-cjk-xelatex
      Downloading jupyter-cjk-xelatex-0.2.tar.gz (1.6 kB)
    Requirement already satisfied: jupyter in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from jupyter-cjk-xelatex) (1.0.0)
    Requirement already satisfied: qtconsole in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from jupyter->jupyter-cjk-xelatex) (4.7.5)
    Requirement already satisfied: ipykernel in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from jupyter->jupyter-cjk-xelatex) (5.3.2)
    Requirement already satisfied: jupyter-console in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from jupyter->jupyter-cjk-xelatex) (6.1.0)
    Requirement already satisfied: nbconvert in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from jupyter->jupyter-cjk-xelatex) (5.6.1)
    Requirement already satisfied: notebook in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from jupyter->jupyter-cjk-xelatex) (6.0.3)
    Requirement already satisfied: ipywidgets in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from jupyter->jupyter-cjk-xelatex) (7.5.1)
    Requirement already satisfied: pygments in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from qtconsole->jupyter->jupyter-cjk-xelatex) (2.6.1)
    Requirement already satisfied: jupyter-client>=4.1 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from qtconsole->jupyter->jupyter-cjk-xelatex) (6.1.6)
    Requirement already satisfied: ipython-genutils in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from qtconsole->jupyter->jupyter-cjk-xelatex) (0.2.0)
    Requirement already satisfied: traitlets in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from qtconsole->jupyter->jupyter-cjk-xelatex) (4.3.3)
    Requirement already satisfied: qtpy in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from qtconsole->jupyter->jupyter-cjk-xelatex) (1.9.0)
    Requirement already satisfied: pyzmq>=17.1 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from qtconsole->jupyter->jupyter-cjk-xelatex) (19.0.1)
    Requirement already satisfied: jupyter-core in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from qtconsole->jupyter->jupyter-cjk-xelatex) (4.6.3)
    Requirement already satisfied: tornado>=4.2 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from ipykernel->jupyter->jupyter-cjk-xelatex) (6.0.4)
    Requirement already satisfied: ipython>=5.0.0 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from ipykernel->jupyter->jupyter-cjk-xelatex) (7.16.1)
    Requirement already satisfied: prompt-toolkit!=3.0.0,!=3.0.1,<3.1.0,>=2.0.0 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from jupyter-console->jupyter->jupyter-cjk-xelatex) (3.0.5)
    Requirement already satisfied: entrypoints>=0.2.2 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from nbconvert->jupyter->jupyter-cjk-xelatex) (0.3)
    Requirement already satisfied: defusedxml in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from nbconvert->jupyter->jupyter-cjk-xelatex) (0.6.0)
    Requirement already satisfied: nbformat>=4.4 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from nbconvert->jupyter->jupyter-cjk-xelatex) (5.0.7)
    Requirement already satisfied: testpath in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from nbconvert->jupyter->jupyter-cjk-xelatex) (0.4.4)
    Requirement already satisfied: jinja2>=2.4 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from nbconvert->jupyter->jupyter-cjk-xelatex) (2.11.2)
    Requirement already satisfied: mistune<2,>=0.8.1 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from nbconvert->jupyter->jupyter-cjk-xelatex) (0.8.4)
    Requirement already satisfied: pandocfilters>=1.4.1 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from nbconvert->jupyter->jupyter-cjk-xelatex) (1.4.2)
    Requirement already satisfied: bleach in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from nbconvert->jupyter->jupyter-cjk-xelatex) (3.1.5)
    Requirement already satisfied: prometheus-client in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from notebook->jupyter->jupyter-cjk-xelatex) (0.8.0)
    Requirement already satisfied: Send2Trash in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from notebook->jupyter->jupyter-cjk-xelatex) (1.5.0)
    Requirement already satisfied: terminado>=0.8.1 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from notebook->jupyter->jupyter-cjk-xelatex) (0.8.3)
    Requirement already satisfied: widgetsnbextension~=3.5.0 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from ipywidgets->jupyter->jupyter-cjk-xelatex) (3.5.1)
    Requirement already satisfied: python-dateutil>=2.1 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from jupyter-client>=4.1->qtconsole->jupyter->jupyter-cjk-xelatex) (2.8.1)
    Requirement already satisfied: six in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from traitlets->qtconsole->jupyter->jupyter-cjk-xelatex) (1.15.0)
    Requirement already satisfied: decorator in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from traitlets->qtconsole->jupyter->jupyter-cjk-xelatex) (4.4.2)
    Requirement already satisfied: pickleshare in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from ipython>=5.0.0->ipykernel->jupyter->jupyter-cjk-xelatex) (0.7.5)
    Requirement already satisfied: pexpect; sys_platform != "win32" in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from ipython>=5.0.0->ipykernel->jupyter->jupyter-cjk-xelatex) (4.8.0)
    Requirement already satisfied: backcall in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from ipython>=5.0.0->ipykernel->jupyter->jupyter-cjk-xelatex) (0.2.0)
    Requirement already satisfied: setuptools>=18.5 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from ipython>=5.0.0->ipykernel->jupyter->jupyter-cjk-xelatex) (49.2.0.post20200714)
    Requirement already satisfied: jedi>=0.10 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from ipython>=5.0.0->ipykernel->jupyter->jupyter-cjk-xelatex) (0.17.1)
    Requirement already satisfied: wcwidth in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from prompt-toolkit!=3.0.0,!=3.0.1,<3.1.0,>=2.0.0->jupyter-console->jupyter->jupyter-cjk-xelatex) (0.2.5)
    Requirement already satisfied: jsonschema!=2.5.0,>=2.4 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from nbformat>=4.4->nbconvert->jupyter->jupyter-cjk-xelatex) (3.0.2)
    Requirement already satisfied: MarkupSafe>=0.23 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from jinja2>=2.4->nbconvert->jupyter->jupyter-cjk-xelatex) (1.1.1)
    Requirement already satisfied: webencodings in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from bleach->nbconvert->jupyter->jupyter-cjk-xelatex) (0.5.1)
    Requirement already satisfied: packaging in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from bleach->nbconvert->jupyter->jupyter-cjk-xelatex) (20.4)
    Requirement already satisfied: ptyprocess>=0.5 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from pexpect; sys_platform != "win32"->ipython>=5.0.0->ipykernel->jupyter->jupyter-cjk-xelatex) (0.6.0)
    Requirement already satisfied: parso<0.8.0,>=0.7.0 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from jedi>=0.10->ipython>=5.0.0->ipykernel->jupyter->jupyter-cjk-xelatex) (0.7.0)
    Requirement already satisfied: attrs>=17.4.0 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from jsonschema!=2.5.0,>=2.4->nbformat>=4.4->nbconvert->jupyter->jupyter-cjk-xelatex) (19.3.0)
    Requirement already satisfied: pyrsistent>=0.14.0 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from jsonschema!=2.5.0,>=2.4->nbformat>=4.4->nbconvert->jupyter->jupyter-cjk-xelatex) (0.16.0)
    Requirement already satisfied: pyparsing>=2.0.2 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from packaging->bleach->nbconvert->jupyter->jupyter-cjk-xelatex) (2.4.7)
    Building wheels for collected packages: jupyter-cjk-xelatex
      Building wheel for jupyter-cjk-xelatex (setup.py) ... [?25ldone
    [?25h  Created wheel for jupyter-cjk-xelatex: filename=jupyter_cjk_xelatex-0.2-py3-none-any.whl size=2076 sha256=7958ac18c463bb7317602ce83b18b6961573c844ee669d2095738ea2ccfaac11
      Stored in directory: /home/ec2-user/.cache/pip/wheels/5e/03/62/b6652316f429b43ac12e266fb32593cb127d6d19ab1a2dff12
    Successfully built jupyter-cjk-xelatex
    Installing collected packages: jupyter-cjk-xelatex
    Successfully installed jupyter-cjk-xelatex-0.2
    [33mWARNING: You are using pip version 20.1.1; however, version 20.2.3 is available.
    You should consider upgrading via the '/home/ec2-user/anaconda3/envs/pytorch_p36/bin/python -m pip install --upgrade pip' command.[0m



```python
!pip install nbconvert

```

    Requirement already satisfied: nbconvert in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (5.6.1)
    Requirement already satisfied: jupyter-core in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from nbconvert) (4.6.3)
    Requirement already satisfied: defusedxml in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from nbconvert) (0.6.0)
    Requirement already satisfied: traitlets>=4.2 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from nbconvert) (4.3.3)
    Requirement already satisfied: bleach in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from nbconvert) (3.1.5)
    Requirement already satisfied: mistune<2,>=0.8.1 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from nbconvert) (0.8.4)
    Requirement already satisfied: pygments in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from nbconvert) (2.6.1)
    Requirement already satisfied: jinja2>=2.4 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from nbconvert) (2.11.2)
    Requirement already satisfied: nbformat>=4.4 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from nbconvert) (5.0.7)
    Requirement already satisfied: entrypoints>=0.2.2 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from nbconvert) (0.3)
    Requirement already satisfied: testpath in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from nbconvert) (0.4.4)
    Requirement already satisfied: pandocfilters>=1.4.1 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from nbconvert) (1.4.2)
    Requirement already satisfied: six in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from traitlets>=4.2->nbconvert) (1.15.0)
    Requirement already satisfied: ipython-genutils in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from traitlets>=4.2->nbconvert) (0.2.0)
    Requirement already satisfied: decorator in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from traitlets>=4.2->nbconvert) (4.4.2)
    Requirement already satisfied: webencodings in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from bleach->nbconvert) (0.5.1)
    Requirement already satisfied: packaging in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from bleach->nbconvert) (20.4)
    Requirement already satisfied: MarkupSafe>=0.23 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from jinja2>=2.4->nbconvert) (1.1.1)
    Requirement already satisfied: jsonschema!=2.5.0,>=2.4 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from nbformat>=4.4->nbconvert) (3.0.2)
    Requirement already satisfied: pyparsing>=2.0.2 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from packaging->bleach->nbconvert) (2.4.7)
    Requirement already satisfied: setuptools in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from jsonschema!=2.5.0,>=2.4->nbformat>=4.4->nbconvert) (49.2.0.post20200714)
    Requirement already satisfied: pyrsistent>=0.14.0 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from jsonschema!=2.5.0,>=2.4->nbformat>=4.4->nbconvert) (0.16.0)
    Requirement already satisfied: attrs>=17.4.0 in /home/ec2-user/anaconda3/envs/pytorch_p36/lib/python3.6/site-packages (from jsonschema!=2.5.0,>=2.4->nbformat>=4.4->nbconvert) (19.3.0)
    [33mWARNING: You are using pip version 20.1.1; however, version 20.2.3 is available.
    You should consider upgrading via the '/home/ec2-user/anaconda3/envs/pytorch_p36/bin/python -m pip install --upgrade pip' command.[0m



```python

```
