## Amazon eCommerce-Project

# Introduction
Amazon is one of the most infamous brands and is often referred to as "one of the most influential economic and cultural forces in the world". 
I recently completed my Google Digital Marketing & E-Commerce Specialization, and I wanted to see how I could utilize what I had recently learned in conjunction with some exploratory data analysis. 

# The Data
This project utilized a sample dataset taken from BrightData. In this dataset there was over 1000 products that included information such as product description, initial price, total reviews, and more. However, there were numerous issues with this data such as:
- Missing Rows
- Diacritics
- Duplicates
- Improper formatting

# Prep-work
Before I could begin data cleaning in Python, I had to first deal with the excel file first. That is, the file was riddle with diacritics which messed with the keyword analysis I was trying to implement.
As such, I found a handy VBA script to get rid of them all.

credit to mpecka/ExcelRemoveAccents.vb

~~~
Function StripAccent(thestring As String)
Dim A As String * 1
Dim B As String * 1
Dim i As Integer
Const AccChars= "áäčďéěíĺľňóôőöŕšťúůűüýřžÁÄČĎÉĚÍĹĽŇÓÔŐÖŔŠŤÚŮŰÜÝŘŽ"
Const RegChars= "aacdeeillnoooorstuuuuyrzAACDEEILLNOOOORSTUUUUYRZ"
For i = 1 To Len(AccChars)
A = Mid(AccChars, i, 1)
B = Mid(RegChars, i, 1)
thestring = Replace(thestring, A, B)
Next
StripAccent = thestring
End Function
~~~

# Preparation
Now that the Excel file was ready, it was time to begin setting up the libraries and the methods I would be using later.

~~~
import pandas as pd
import seaborn as sns
import plotly.express as px
import numpy as np
import string
import unicodedata
import nltk
from nltk.tokenize import word_tokenize
from nltk.stem import WordNetLemmatizer
from wordcloud import WordCloud
from collections import Counter
from sklearn.linear_model import LinearRegression

def check(df):
    list=[]
    for col in df.columns:
        columns = df.columns
        dtype = df[col].dtypes
        instances = df[col].count()
        unique = df[col].nunique()
        sum_null = df[col].isnull().sum()
        duplicates = df[col].duplicated().sum()
        list.append([dtype,instances,unique,sum_null,duplicates])
    data_check = pd.DataFrame(list,columns=["dtype","instances","unique","sum_null","duplicates"],index=df.columns)
    
    return data_check

def check_unique(df):
    nunique=df.apply(lambda col: col.nunique())
    unique_values = df.apply(lambda col: col.unique())
    data_check = pd.DataFrame({'uni_count': nunique, 'unique_values': unique_values})
    
    return data_check

def check_missing(df):
    values = df.isnull().sum()
    percentage = (values/len(df)) * 100
    missing_df = pd.DataFrame({'Missing Values': values, 'Percentage (%)': percentage})
    
    return missing_df[missing_df['Missing Values'] > 0].sort_values(by='Percentage (%)', ascending=False) 

def detect_outliers(dataframe, column):
    Q1 = dataframe[column].quantile(0.25)
    Q3 = dataframe[column].quantile(0.75)
    IQR = Q3 - Q1
    lower_bound = Q1 - 1.5 * IQR
    upper_bound = Q3 + 1.5 * IQR
    
    return dataframe[(dataframe[column] < lower_bound) | (dataframe[column] > upper_bound)]

def drop_outliers(dataframe, column): 
    Q1 = dataframe[column].quantile(0.25)
    Q3 = dataframe[column].quantile(0.75)
    IQR = Q3 - Q1
    lower_bound = Q1 - 1.5*IQR
    upper_bound = Q3 + 1.5*IQR
 
    upper_array = np.where(dataframe[column] >= upper_bound)[0]
    lower_array = np.where(dataframe[column] <= lower_bound)[0]
 
    dataframe.drop(index=upper_array, axis=1, inplace=True)
    dataframe.drop(index=lower_array, axis=1, inplace=True)

def clean_texts(text):
    text = text.lower()
    text = text.translate(str.maketrans("", "", exclist))
    text = text.replace("leased", "lease")
    tokens = word_tokenize(text)
    tokens = [lemmatizer.lemmatize(token) for token in tokens]
    tokens = [token for token in tokens if token not in stop_words]
    clean_text = " ".join(tokens)
    
    return clean_text
~~~

# Data Cleaning and Exploration
In this phase we took a more in-depth look at our data to get a sense of what we were dealing with.
![Overview_check](https://github.com/ThisIsDNa/Amazon-eCommerce-Project/assets/42982734/b8da152a-e3d9-409a-9373-687b90a693b8)
![Unique_Check](https://github.com/ThisIsDNa/Amazon-eCommerce-Project/assets/42982734/78b55ed0-08f3-4030-b245-134ee5c64fc0)
![Check_for_missing_values](https://github.com/ThisIsDNa/Amazon-eCommerce-Project/assets/42982734/1c0402d0-5676-4e50-ac7b-469d22c7823f)

After our exploration, we decided to take a standard approach to cleaning up our data by:
1. Removing unnecessary data columns
2. Detecting and handling missing values
3. Identifying and removing outliers

Reasons for dropping columns
- currency
  > Everything is in USD already, so this is redundant
- asin, parent_asin, input_asin
  > We aren't using asin for Data Analysis or Keyword Analysis, so we'll drop it
- domain
  > Everything is hosted on Amazon.com, so this is redundant
- url, image_url, image
  > In another excercise it makes sense to run keyword analysis, but for this project we're just going to focus on 
    descriptions and top_review
- item_weight, product_dimensions, seller_id, model_number, upc, variations, features, buybox_prices
  > Unnecessary
- final_price_high, format
  > Blank, so we'll just remove it
- department
  > There's over 75% missing rows so it's fine to exclude
~~~
df.drop(["department"],axis=1,inplace=True)
df.drop(["currency",
         "asin",
         "parent_asin",
         "input_asin",
         "domain",
         "url",
         "image_url",
         "image",
         "item_weight",
         "product_dimensions",
         "seller_id",
         "model_number",
         "upc",
         "variations",
         "features",
         "buybox_prices",
         "final_price_high",
         "format"],axis=1,inplace=True)
~~~

# Handling Missing Values
In order to deal with the missing values in our columns we will apply the following fixes:
1. Replace missing values in categorical columns with the placeholder "Unknown"
2. Replace missing values in numerical columns with the mean of that column

~~~
categorical_columns = df.select_dtypes(include=['object']).columns
numerical_columns = df.select_dtypes(exclude=['object']).columns

for column in categorical_columns:
    df[column].fillna('Unknown', inplace=True)

for column in numerical_columns:
    mean_value = df[column].mean()
    df[column].fillna(mean_value, inplace=True)
~~~


# Handling Outliers
In order to deal with outliers, we decied to utilize the Interquartile Range method. The first step included identifying said outliers, which would then be removed by a later formula.

~~~
outliers_data = {}
for column in numerical_columns:
    outliers = detect_outliers(df, column)
    outliers_data[column] = len(outliers)

outliers_data
~~~

![Outliers_Check](https://github.com/ThisIsDNa/Amazon-eCommerce-Project/assets/42982734/e15eab39-e396-42a2-bb4c-a87d1ad2d7b9)

~~~
Q1 = df['reviews_count'].quantile(0.25)
Q3 = df['reviews_count'].quantile(0.75)
IQR = Q3 - Q1
lower = Q1 - 1.5*IQR
upper = Q3 + 1.5*IQR

upper_array = np.where(df['reviews_count'] >= upper)[0]
lower_array = np.where(df['reviews_count'] <= lower)[0]
 
df.drop(index=upper_array, axis=1, inplace=True)
df.drop(index=lower_array, axis=1, inplace=True)
~~~
Quick peek into cleaned dataset
![Capture](https://github.com/ThisIsDNa/Amazon-eCommerce-Project/assets/42982734/5088a645-3fb6-4e0e-9da0-5c6e0fd97674)


# Data Mining
In this phase, we'll explore the data to uncover patterns and insights. This involves:

1. Descriptive Statistics : Understanding the basic overview of our data
3. Visualizations: Using plots to understand the distribution, relationships, and patterns in data
4. Analysis: Utilizing statistical techniques to derive deeper trends and insights

~~~
descriptive_stats = df.describe(include=[float, int])
descriptive_stats
~~~

![descriptive_statistics](https://github.com/ThisIsDNa/Amazon-eCommerce-Project/assets/42982734/a32de43a-a252-4465-9565-8cc152f5c936)

- inital_price: There is a large range when it comes to price, with the cheapest being $3.16 and the most expensive sitting at $1535.95
- reviews_count: The average amount of reviews per product sits around 3.57
- number_of_sellers: There is a large range when it comes to number of sellers, with as many as 43 different sellers for a single product
- answered_questions: On average, there is about 4.79 answered questions per product
- images_count: On average, there are 1.93 images per product
- video_count: On average, there are 0.29 videoes per product
- rating: The average rating of a typical product in this dataset sits around 3.47
- discount: The average discount of a typical product is 11%
- final_price: Similar to the initial price, there is a large range with the cheapest being $0.29 and the most expensive at $3480.75

Next, we'll use visualizations to get a better understanding of the data's distribution and relationships:
- Heatmap to identify variables that are strongly correlated with Rating
- Distribution of Product Ratings
- Relationship between ratings and reviews count
- Relationship between ratings and answered questions

![Amazon_eCommerce_Heatmap](https://github.com/ThisIsDNa/Amazon-eCommerce-Project/assets/42982734/d5adbd38-7ab6-4a5a-971b-4b446349a2f7)
![Amazon_eCommerce_DistributionOfProductRatings](https://github.com/ThisIsDNa/Amazon-eCommerce-Project/assets/42982734/73aac339-2498-437c-899d-ff9a32216d58)
![Amazon_eCommerce_Rating_Reviews_Regression](https://github.com/ThisIsDNa/Amazon-eCommerce-Project/assets/42982734/b5ac9c3a-1636-43dc-aba1-b01ed304b296)
![Amazon_eCommerce_Rating_Answered_Questions_Regression](https://github.com/ThisIsDNa/Amazon-eCommerce-Project/assets/42982734/f27ab2c4-7dcd-49fd-be71-1a714e8e4a5b)

From our analysis so far:

- Relationship between Ratings and Reviews: There seems to be a positive correlation between highly rated products and the number of reviews assocaited with them. This suggests that highly rated products tend to attract more reviews.
- Relationship between Ratings and Answered Questions: There seems to be a positive correlation between high highly rated products and answered questions as well. This suggests that while it does not have as much of an influence as number of reviews, the more answered questions a product has the more positively viewed it becomes.

# Keyword Preparation

So this is normally where my exploratory data analysis would end, but I was inspired by Google's eCommerce course to take a look into Keyword Analysis. However, just like working with any other dataset, it was imperative of me to clean the data I would be analyzing and help set it up for success.

In order to better analyze the data, there were a series of steps that needed to be accomplished:
1. Convert all words to lower cases
2. Remove punctuations an dnumbers
3. Tokenize texts to words so that machines can work with them
4. Remove stop words
5. Replacing words with its most basic form 

~~~
# Instantiate
lemmatizer = WordNetLemmatizer()
# Create our own stop words
stop_words = ("0o", "0s", "3a", "3b", "3d", "6b", "6o", 
              "a", "a1", "a2", "a3", "a4", "ab", "able", 
              "about", "above", "abst", "ac", "accordance", "according", 
              "accordingly", "across", "act", "actually", "ad", "added", 
              "adj", "ae", "af", "affected", "affecting", "affects", "after", 
              "afterwards", "ag", "again", "against", "ah", "ain", "ain't", 
              "aj", "al", "all", "allow", "allows", "almost", "alone", "along", 
              "already", "also", "although", "always", "am", "among", "amongst", 
              "amoungst", "amount", "an", "and", "announce", "another", "any", 
              "anybody", "anyhow", "anymore", "anyone", "anything", "anyway", 
              "anyways", "anywhere", "ao", "ap", "apart", "apparently", "appear", 
              "appreciate", "appropriate", "approximately", "ar", "are", "aren", 
              "arent", "aren't", "arise", "around", "as", "a's", "aside", "ask", 
              "asking", "associated", "at", "au", "auth", "av", "available", "aw", 
              "away", "awfully", "ax", "ay", "az", "b", "b1", "b2", "b3", "ba", "back", 
              "bc", "bd", "be", "became", "because", "become", "becomes", "becoming", "been", 
              "before", "beforehand", "begin", "beginning", "beginnings", "begins", "behind", 
              "being", "believe", "below", "beside", "besides", "best", "better", "between", 
              "beyond", "bi", "bill", "biol", "bj", "bk", "bl", "bn", "both", "bottom", "bp", 
              "br", "brief", "briefly", "bs", "bt", "bu", "but", "bx", "by", "c", "c1", "c2", 
              "c3", "ca", "call", "came", "can", "cannot", "cant", "can't", "cause", "causes", "cc", "cd", 
              "ce", "certain", "certainly", "cf", "cg", "ch", "changes", "ci", "cit", "cj", "cl", 
              "clearly", "cm", "c'mon", "cn", "co", "com", "come", "comes", "con", "concerning", 
              "consequently", "consider", "considering", "contain", "containing", "contains", 
              "corresponding", "could", "couldn", "couldnt", "couldn't", "course", "cp", "cq", 
              "cr", "cry", "cs", "c's", "ct", "cu", "currently", "cv", "cx", "cy", "cz", "d", "d2",
              "da", "date", "dc", "dd", "de", "definitely", "describe", "described", "despite", "detail", 
              "df", "di", "did", "didn", "didn't", "different", "dj", "dk", "dl", "do", "does", "doesn", 
              "doesn't", "doing", "don", "done", "don't", "down", "downwards", "dp", "dr", "ds", "dt", "du", 
              "due", "during", "dx", "dy", "e", "e2", "e3", "ea", "each", "ec", "ed", "edu", "ee", "ef", 
              "effect", "eg", "ei", "eight", "eighty", "either", "ej", "el", "eleven", "else", "elsewhere",
              "em", "empty", "en", "end", "ending", "enough", "entirely", "eo", "ep", "eq", "er", "es", 
              "especially", "est", "et", "et-al", "etc", "eu", "ev", "even", "ever", "every", "everybody", 
              "everyone", "everything", "everywhere", "ex", "exactly", "example", "except", "ey", "f", "f2", 
              "fa", "far", "fc", "few", "ff", "fi", "fifteen", "fifth", "fify", "fill", "find", "fire", "first", 
              "five", "fix", "fj", "fl", "fn", "fo", "followed", "following", "follows", "for", "former", "formerly", 
              "forth", "forty", "found", "four", "fr", "from", "front", "fs", "ft", "fu", "full", "further", 
              "furthermore", "fy", "g", "ga", "gave", "ge", "get", "gets", "getting", "gi", "give", "given", 
              "gives", "giving", "gj", "gl", "go", "goes", "going", "gone", "got", "gotten", "gr", "greetings", 
              "gs", "gy", "h", "h2", "h3", "had", "hadn", "hadn't", "happens", "hardly", "has", "hasn", "hasnt", 
              "hasn't", "have", "haven", "haven't", "having", "he", "hed", "he'd", "he'll", "hello", "help", 
              "hence", "her", "here", "hereafter", "hereby", "herein", "heres", "here's", "hereupon", "hers", 
              "herself", "hes", "he's", "hh", "hi", "hid", "him", "himself", "his", "hither", "hj", "ho", 
              "home", "hopefully", "how", "howbeit", "however", "how's", "hr", "hs", "http", "hu", "hundred", 
              "hy", "i", "i2", "i3", "i4", "i6", "i7", "i8", "ia", "ib", "ibid", "ic", "id", "i'd", "ie", "if", "ig", 
              "ignored", "ih", "ii", "ij", "il", "i'll", "im", "i'm", "immediate", "immediately", "importance", 
              "important", "in", "inasmuch", "inc", "indeed", "index", "indicate", "indicated", "indicates", 
              "information", "inner", "insofar", "instead", "interest", "into", "invention", "inward", "io", 
              "ip", "iq", "ir", "is", "isn", "isn't", "it", "itd", "it'd", "it'll", "its", "it's", "itself", 
              "iv", "i've", "ix", "iy", "iz", "j", "jj", "jr", "js", "jt", "ju", "just", "k", "ke", "keep", 
              "keeps", "kept", "kg", "kj", "km", "know", "known", "knows", "ko", "l", "l2", "la", "largely", 
              "last", "lately", "later", "latter", "latterly", "lb", "lc", "le", "least", "les", "less", "lest", 
              "let", "lets", "let's", "lf", "like", "liked", "likely", "line", "little", "lj", "ll", "ll", "ln", 
              "lo", "look", "looking", "looks", "los", "lr", "ls", "lt", "ltd", "m", "m2", "ma", "made", "mainly", 
              "make", "makes", "many", "may", "maybe", "me", "mean", "means", "meantime", "meanwhile", "merely", 
              "mg", "might", "mightn", "mightn't", "mill", "million", "mine", "miss", "ml", "mn", "mo", "more", 
              "moreover", "most", "mostly", "move", "mr", "mrs", "ms", "mt", "mu", "much", "mug", "must", "mustn", 
              "mustn't", "my", "myself", "n", "n2", "na", "name", "namely", "nay", "nc", "nd", "ne", "near", 
              "nearly", "necessarily", "necessary", "need", "needn", "needn't", "needs", "neither", "never", 
              "nevertheless", "new", "next", "ng", "ni", "nine", "ninety", "nj", "nl", "nn", "no", "nobody", 
              "non", "none", "nonetheless", "noone", "nor", "normally", "nos", "not", "noted", "nothing", "novel", 
              "now", "nowhere", "nr", "ns", "nt", "ny", "o", "oa", "ob", "obtain", "obtained", "obviously", "oc", 
              "od", "of", "off", "often", "og", "oh", "oi", "oj", "ok", "okay", "ol", "old", "om", "omitted", "on", 
              "once", "one", "ones", "only", "onto", "oo", "op", "oq", "or", "ord", "os", "ot", "other", "others", 
              "otherwise", "ou", "ought", "our", "ours", "ourselves", "out", "outside", "over", "overall", "ow", 
              "owing", "own", "ox", "oz", "p", "p1", "p2", "p3", "page", "pagecount", "pages", "par", "part", 
              "particular", "particularly", "pas", "past", "pc", "pd", "pe", "per", "perhaps", "pf", "ph", "pi", 
              "pj", "pk", "pl", "placed", "please", "plus", "pm", "pn", "po", "poorly", "possible", "possibly", 
              "potentially", "pp", "pq", "pr", "predominantly", "present", "presumably", "previously", "primarily", 
              "probably", "promptly", "proud", "provides", "ps", "pt", "pu", "put", "py", "q", "qj", "qu", "que", 
              "quickly", "quite", "qv", "r", "r2", "ra", "ran", "rather", "rc", "rd", "re", "readily", "really", 
              "reasonably", "recent", "recently", "ref", "refs", "regarding", "regardless", "regards", "related", 
              "relatively", "research", "research-articl", "respectively", "resulted", "resulting", "results", "rf", 
              "rh", "ri", "right", "rj", "rl", "rm", "rn", "ro", "rq", "rr", "rs", "rt", "ru", "run", "rv", "ry", 
              "s", "s2", "sa", "said", "same", "saw", "say", "saying", "says", "sc", "sd", "se", "sec", "second", 
              "secondly", "section", "see", "seeing", "seem", "seemed", "seeming", "seems", "seen", "self", "selves", 
              "sensible", "sent", "serious", "seriously", "seven", "several", "sf", "shall", "shan", "shan't", "she", 
              "shed", "she'd", "she'll", "shes", "she's", "should", "shouldn", "shouldn't", "should've", "show", "showed", 
              "shown", "showns", "shows", "si", "side", "significant", "significantly", "similar", "similarly", "since", 
              "sincere", "six", "sixty", "sj", "sl", "slightly", "sm", "sn", "so", "some", "somebody", "somehow", 
              "someone", "somethan", "something", "sometime", "sometimes", "somewhat", "somewhere", "soon", "sorry", 
              "sp", "specifically", "specified", "specify", "specifying", "sq", "sr", "ss", "st", "still", "stop", 
              "strongly", "sub", "substantially", "successfully", "such", "sufficiently", "suggest", "sup", "sure", 
              "sy", "system", "sz", "t", "t1", "t2", "t3", "take", "taken", "taking", "tb", "tc", "td", "te", "tell", 
              "ten", "tends", "tf", "th", "than", "thank", "thanks", "thanx", "that", "that'll", "thats", "that's", 
              "that've", "the", "their", "theirs", "them", "themselves", "then", "thence", "there", "thereafter", 
              "thereby", "thered", "therefore", "therein", "there'll", "thereof", "therere", "theres", "there's", 
              "thereto", "thereupon", "there've", "these", "they", "theyd", "they'd", "they'll", "theyre", "they're", 
              "they've", "thickv", "thin", "think", "third", "this", "thorough", "thoroughly", "those", "thou", 
              "though", "thoughh", "thousand", "three", "throug", "through", "throughout", "thru", "thus", "ti", 
              "til", "tip", "tj", "tl", "tm", "tn", "to", "together", "too", "took", "top", "toward", "towards", 
              "tp", "tq", "tr", "tried", "tries", "truly", "try", "trying", "ts", "t's", "tt", "tv", "twelve", 
              "twenty", "twice", "two", "tx", "u", "u201d", "ue", "ui", "uj", "uk", "um", "un", "under", "unfortunately", 
              "unless", "unlike", "unlikely", "until", "unto", "uo", "up", "upon", "ups", "ur", "us", "use", "used", 
              "useful", "usefully", "usefulness", "uses", "using", "usually", "ut", "v", "va", "value", "various", "vd", 
              "ve", "ve", "very", "via", "viz", "vj", "vo", "vol", "vols", "volumtype", "vq", "vs", "vt", "vu", "w", 
              "wa", "want", "wants", "was", "wasn", "wasnt", "wasn't", "way", "we", "wed", "we'd", "welcome", "well", 
              "we'll", "well-b", "went", "were", "we're", "weren", "werent", "weren't", "we've", "what", "whatever", 
              "what'll", "whats", "what's", "when", "whence", "whenever", "when's", "where", "whereafter", "whereas", 
              "whereby", "wherein", "wheres", "where's", "whereupon", "wherever", "whether", "which", "while", "whim", 
              "whither", "who", "whod", "whoever", "whole", "who'll", "whom", "whomever", "whos", "who's", "whose", 
              "why", "why's", "wi", "widely", "will", "willing", "wish", "with", "within", "without", "wo", "won", 
              "wonder", "wont", "won't", "words", "world", "would", "wouldn", "wouldnt", "wouldn't", "www", "x", 
              "x1", "x2", "x3", "xf", "xi", "xj", "xk", "xl", "xn", "xo", "xs", "xt", "xv", "xx", "y", "y2", 
              "yes", "yet", "yj", "yl", "you", "youd", "you'd", "you'll", "your", "youre", "you're", "yours", 
              "yourself", "yourselves", "you've", "yr", "ys", "yt", "z", "zero", "zi", "zz", 
              "product", "design", "tool", "type", "material", "ha", "feature", "size", "color", "inch", "mm", "unknown",
              "cttw", "ring", "stone", "silver", "zirconia", "sterling")
exclist = string.punctuation + string.digits
df['description'] = df['description'].apply(clean_texts)
df['top_review'] = df['top_review'].apply(clean_texts)
~~~~

# Extract Keywords
Now for the fun part. When I was researching Keyword Analysis, there were two main visualizations that stood out to me that would help gather insights and identify trends.
Before creating these visualizations, I decided that taking a deeper look into a specific set of products would be more useful than looking at the dataset as a whole.
So, for the purpose of this excercise, I decided to take a deeper dive into that keywords are associated with Clothing products on Amazon's store.

The first visualization I decided to go with was a wordcloud.
![Amazon_Product_Description_WordCloud](https://github.com/ThisIsDNa/Amazon-eCommerce-Project/assets/42982734/6fb78840-a7e4-484d-bb61-845d19cc96f9)


Several observations:

When it comes to clothing, the biggest keywords that stood out to me were:
1. woman
2. style
3. dress
4. sweatshirt
5. fit

Similar to the word cloud, we can infer that the top rated products have the following keywords associated with them by displaying them on a frequency chart
![Amazon_Clothing_Product_Description_Bar_Chart](https://github.com/ThisIsDNa/Amazon-eCommerce-Project/assets/42982734/8d9e780f-31af-4feb-8480-e6047b8c9326)

To summarize, the top rated Clothing products on the Amazon Store are:

- targeted towards women
- the most popular styles of clothing are pajamas, dresses, pants, scrubs, and sweatshirts
- important factors of these clothings include comfort, quality, fit, softness, and pockets

# Conclusion

From the results of the analysis, it is evident that there are clear factors for successful products on Amazon's store. To start, highly rated products have more customers taking the time to post reviews and customers seem to prefer products that take the time to answer their questions as well. Furthermore, from the Keyword Analysis for Clothing products in particular, female customers are interested in specific styles of clothing such as pajamas and dresses that can be associated with the words comfort, quality, and fit. Moving forward, if I were to provide recommendations to potential companies that are looking to introduce new clothing articles to Amazon I would recommend the following:

- Take the time to respond to customer's questions
- When it comes to Clothing, Females are more interested in specific products: pajamas, dresses, and sweatshirts
- Female customers are more receptive to keywords such as comfort, quality, and fit

In a real-world scenario, I would be utilizing these insights I have found to help drive marketing strategies and develop new products/services. In the future, I would like to explore more relationships in the dataset such as how much of an effect Discounts has on Product Ratings, or if Free Delivery is important.

