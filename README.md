# Hackathon
Babe got Bytes problem 1
import spacy
import time
import re
from bs4 import BeautifulSoup
import requestsexz
import csv
from newsapi import NewsApiClient
from textblob import TextBlob
import pandas as pd
from datetime import datetime
from datetime import timezone
import justext
import matplotlib.pyplot as plt
import sys


nlp= spacy.load('en_core_web_sm')

def create_bar_graph():
    df = pd.read_csv("Analytics.csv")
    ax = df.plot(kind='bar', y='Score',rot=0,color='blue')
    plt.xlabel('Article number')
    plt.ylabel('Score')

    plt.title('Comparison of all the articles')


    plt.show()

def create_line_graph():
    df = pd.read_csv("Analytics.csv")

    # Create a line graph
    df.plot(x='DateTime', y='Score', kind='line',figsize=(20, 4))

    # Add a title and axis labels
    plt.title("Line Graph Title")
    plt.xlabel("Timeline")
    plt.ylabel("Score")

    # Display the chart
    plt.show()

def create_pie_chart():
    df = pd.read_csv("Analytics.csv")
    colors = ["#1f77b4", "#ff7f0e", "#2ca02c", "#d62728", "#8c564b"]
    explode = (0.1,0,0,0)

    # Create a pie chart using the value_counts method
    df['effect'].value_counts().plot.pie(autopct='%1.1f%%', colors=colors,shadow=True, startangle=140)

    # Add a title
    plt.title("Pie Chart Title")

    # Display the chart
    plt.show()
    
def download_data(url):
#     print("starting download")
#     r = requests.get(url)
#     htmlContent = r.content
#     soup = BeautifulSoup(htmlContent, 'html.parser')
# #     data = soup.find_all('p')
#     print(soup.find_all('p'))
#     HTMLParser = htmlcontent
#     data = HTMLParser.handle_data()
    response = requests.get(url)
    
    paragraphs = justext.justext(response.content, justext.get_stoplist("English"))
    for paragraph in paragraphs:
        if(len(paragraph) > 10):
            file = open("All_articles.txt" , "a+" , encoding = "utf-8")
            file.writelines(paragraph.text)
            file.write("________________________________________________________________________________")
            file.close()
    
#     print(url)
#     print("data written")

def review_to_company(article_text):
    # Process the article text with spaCy
    doc = nlp(article_text)

    reputation_words = ['bad', 'poor quality', 'poor customer service']
    layoff_words = ['laid off','mass layoff','layoff']
    financial_words=['bankruptcy','bankrupt','financial crisis']
    legal_words=['legal','allegations','legal battle','illegal','investigation']
    # Initialize the PhraseMatcher with the spaCy model's vocabulary
    matcher = PhraseMatcher(nlp.vocab)

    # Create patterns for the positive and negative words
    reputation_patterns = [nlp(word) for word in reputation_words]
    layoff_patterns= [nlp(word) for word in layoff_words ]
    financial_patterns=[nlp(word) for word in financial_words]
    legal_patterns=[nlp(word) for word in legal_words]

    # Add the patterns to the matcher
    matcher.add("Reputation", None, *reputation_patterns)
    matcher.add("Layoff", None, *layoff_patterns)
    matcher.add("Financial", None, *financial_patterns)
    matcher.add("Legal", None, *legal_patterns)

    # Use the matcher to find matches in the doc
    matches = matcher(doc)

    for match_id, start, end in matches:
        if nlp.vocab.strings[match_id] == "Reputation":
            return"Reputation of company is threatened"
            break

        elif nlp.vocab.strings[match_id] == "Layoff":
            return"Company is threatened by Layoff"
            break

        elif nlp.vocab.strings[match_id] == "Financial":
            return"Company is having financial downfall"
            break
        elif nlp.vocab.strings[match_id] == "Legal":
            return"Company is having legal issues"
            break
        else:
            return"Other threats"
            break

#sentiment analysis
def sentiment(url):
#     response = requests.get(url)
    article_text = (requests.get(url)).text
    article_blob = TextBlob(article_text)
    sentiment = article_blob.sentiment.polarity
    sentiment  = round(sentiment , 3)
    q= quality(sentiment)
    if(q == "Negative" or q == "Threat"):
        r = review_to_company(article_text)
    else:
        r = "None"
#     print("here")
    download_data(url)
#     print("returning sentiment")
    return [sentiment , q, r]
def quality(sentiment):
    if sentiment<=-0.3:
        return "Threat"
    elif sentiment>-0.3 and sentiment < 0:
        return "Negative"
    elif sentiment > 0 and sentiment < 0.3:
        return "Positive"
    elif sentiment>=0.3:
        return "OP"
    else:
        return " Neutral"

#checking the urls
def is_valid(query , titleList):
    valid_titles= []
    docs =[nlp(title) for title in titleList]
    for doc in docs:
        for ent in doc.ents:
            if(ent.text == query and ent.label_ == 'ORG'):
                valid_titles.append(str(doc))
                break
            else:
                continue
    return valid_titles
#getting news urls
newsapi = NewsApiClient(api_key='12fc75e95e1a4a93b72fd129fb375f55')
query = input("enter the company name you want to search : ")
all_urls = []
all_titles = []
all_articles = newsapi.get_everything(q= query,
                                      from_param='2023-01-01',
                                      to='2023-01-13',
                                      language='en',
                                      sort_by='popularity')

for i in range(len(all_articles['articles'])):
    all_urls.append(all_articles['articles'][i]['url'])
    all_titles.append(all_articles['articles'][i]['description'])

required_titels = []
required_titles = is_valid(query , all_titles)
# print("--------required titles--------------")
# print(required_titles)
required_url = []
upload_date=[]
for i in range(len(required_titles)):
    for j in range(len(all_articles['articles'])):
        if(all_articles['articles'][j]['description'] == required_titles[i]):
            required_url.append(all_articles['articles'][j]['url'])
            d = datetime.fromisoformat((all_articles['articles'][j]['publishedAt'])[:-1])
            d.strftime('%Y-%m-%d')
            upload_date.append(d)
            
        else:
            continue
# print("--------required url--------------")
# print(required_url)        
# and all_articles['articles'][i]['title'] == required_titles[i])
with open('example.csv' , 'w' , encoding = "utf-8") as file:
    writer = csv.writer(file)
    writer.writerow(required_url)

if (not(required_url)):
    print("There were no articles published for this company this month \nBetter Luck Next Time :(")
    print("-------Exiting--------")
    sys.exit(0)
sent =[]
all_sentiment_score =[]
all_impressions = []
all_problems = []
for url in required_url:
    try:
        sent = sentiment(url)
#         print("sentiment score : " , sent[0])
        all_sentiment_score.append(sent[0])
#         print("the impression of the article is :", sent[1])
        all_impressions.append(sent[1])
#         print("the major problem seems to be :", sent[2])
        all_problems.append(sent[2])
    except:
        continue
df = pd.DataFrame()
df["DateTime"] = upload_date
df["Score"] = pd.Series(all_sentiment_score)
df["effect"] = pd.Series(all_impressions)
df["problems"] = pd.Series(all_problems)
df['URL'] = pd.Series(required_url)
# print(df)
df.to_csv("Analytics.csv")

create_line_graph()
create_pie_chart()
create_bar_graph()
