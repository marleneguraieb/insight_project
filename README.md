# Write Right!
## Helping CrossLead identify successful  business objectives

This post describes my project as a Fellow in Insight Data Science, a program that helps academics make the transition to industry. At Insight I consulted with [CrossLead](https://www.crosslead.com), an enterprise SaaS platform that provides a leadership and management system to optimize organizational performance. They help companies organize their business strategies, principles, and goals into a hierarchical structure that helps them track progress and link objectives across different teams and levels of their organization. This blog post describes some of the insights that we got by building a model that could identify and predict success of business objectives. You can watch my [slides here](https://docs.google.com/presentation/d/1uPaN-WuReAYp2LgnWXhzgJlLVHGPwB7GhRvLBIjUV9c/pub?start=false&loop=false&delayms=5000), or keep reading for a more detailed story. 

---

# Contents

[**1. CrossLead's Challenge?**](#cross_challenge)

[**2. Defining the target**](#def_target)

[**3. First pass: BoW Model**](#first_pass)

[**4. Can this be improved?**](#second_pass)

[**5. How accurate is it?**](#accuracy)

[**6. So, how should we write?**](#how_write)

[**7. What's next?**](#what_next)

---

# <a name="cross_challenge">1. CrossLead's Challenge</a>

CrossLead’s management model uses an Alignment Triangle, in which the company’s vision stands at the top, and then progressively more concrete ways of attaining it are stacked hierarchically below. They currently provide a platform their clients to upload their business plan and track the success of individual business objectives based on this model. This structure nudges their clients towards a framework (within their platform) in which smaller, more concrete initiatives are linked to more comprehensive strategies and then to the company’s mission and vision. They also provide a dashboard and some analytics that lets them track the progress of each goal and link the team members responsible for it. Thus, a fictitious company, A Inc. would interact with CrossLead's platform in this way:

<p align="center">
  <img src="https://www.dropbox.com/s/k8dyq30tbpfkiih/Screenshot%202017-02-09%2014.58.04.png?raw=1" />
</p>

One of Crosslead’s own goals is to expand their analytic capabilities to provide insights to their clients on how to optimize their strategic planning approach. In support of this, they asked me to develop a predictive model for objective completion using the information contained in the description of the objective itself. Predicting objective completion is a way for CrossLead to understand user engagement on their platform. In this way, predicting objective completion can help account managers at the company better understand their own clients' use of the software product.  

Management best practices recommend that companies state their goals in a way that is more specific, measurable and timed, but can we find this in CrossLead’s data? How much information can we extract from the way an objective is written that gives us generalizable insights about how other companies should structure their plans?

Quite a bit, it turns out…

# <a name="def_target">2. Defining the target</a>

CrossLead has around 7,500 business objectives in their dataset. These objectives all have a description (like the mock one above). Around 1/3 of them have never been updated, roughly 1/3 have been updated, but there has been no progress, and the rest have a level of completion between 0 and 100%. I will divide these values into four classes: the first one `forgotten` fot the objectives that have never been updated; then `zero` for the ones that have been updated but which have 0% progress; the third one -the `half or less` class- encompasses objectives with progress up to 50%; and the `more than half` class has objectives which are more than 50% complete. For CrossLead, it is especially important to have a classifier that can discriminate between objectives that have some level of completion and those that are `forgotten` or `zero`, given that -for any level of success attained by the client- if they are updating their objective they are engaging with the platform. 

# <a name="first_pass">3. First pass: BoW model</a>

Stating business objectives usually involves answers to three of the five W questions: **WHO**, **WHAT**, and **WHEN**. Intuitively good goals will find a way to succinctly answer those questions in a short sentence. However, analyzing the structure of text written for this specific purpose poses a particular challenge for NLP. The *whens* will mostly be dates, the *whats* will include a lot of numbers and named entities and the *whos* will be mostly named entities: teams, company names, organizations.  

NLP techniques like Tf-Idf would excel at assigning importance to the named entities in the objectives since their frequency on the overall corpus is low. However, a transformation such as this would inform the classification algorithm about the entities that are most likely to be associated with completed objectives. This is not CrossLead's challenge, as they would like to predict which types of objectives are more thoroughly completed. 

In order to address this, the first task is to perform entity recognition on the corpus to replace these entities with a tag so that the structure of the text is preserved, but not the particulars of the team or date in the text. For this I use [SpaCy's](https://spacy.io) built in entity recognition tool and replace (in the tagged text) all numbers and dates with their entity code. As for particular team names, the ideal thing would be to train a model for entity recognition, but since it would require manual coding, a workaround is to tag the text and replace all proper nouns with a code. So, a fictitious objective would be preprocessed as follows before applying Tf-Idf:

Original:

>Identify 8 key business partners for Team Alpha to propose microblogging deal by end of 2016.


Entities and proper nouns:

>Identify **NUM** key business partners for **PROPN** to propose microblogging deal by **DATE**


Stopwords and lemmatizing:

>Identify **NUM** key business partner **PROPN** propose microblogging deal **DATE**

Once the text has been processed, I use Tf-Idf to vectorize the feature space taking the collection of objective descriptions as a corpus and each individual description as a document. In this case, most of the work is being done by the inverse document frequency component of Tf-Idf, given that short sentences are unlikely to have words repeated more than once (Tf will be 1 for most of the words). While this might seem odd, the feature space produced by this transformation is the best way to find *uniqueness* of the words in the objective description (like in other short text contexts -e.g. finding the relevance of search terms or tweets).

<p align="center">
  <img src="https://www.dropbox.com/s/v61u51pnwnk3da4/Screenshot%202017-02-09%2014.54.15.png?raw=1" />
</p>


Once the features have been vectorized, the next task is to apply a classification algorithm to predict the class of each objective. The choice of a classifier has to take into account that interpretability is one of the main objectives since we want to be able to construct a style editor from the feature importance analysis. This is the main reason why I chose an SVM classifier (with C=1) with a linear kernel, since it allows me to extract the features (in this case bi-grams and words) with the highest *coefficients* to analyze the words that were most predictive of each particular class. The model achieves an accuracy of 65%. Experimenting with different kernels and doing grid search can boost the accuracy up around 1%, but performing feature importance analysis with non linear transformation is not straightforward given that we do not know the mapping function explicitly. 

# <a name="second_pass">4. Can this be improved?</a>

The bag of words approach discussed above performs reasonably well (if we were to predict only the most common class I would get around 33% accuracy), but there is still more information in the text that I haven't made use of. Information about the length of the sentences or the specificity of the words can also be used to improve performance. To do this, I use SpaCy's parser to extract the syntactic properties of words. I extract syntactic features such as the length of the longest sub-tree in the parsed sentence, the percentage of the words that are named entities, proper nouns and numbers; as well as the percentage of words out of the English vocabulary (to find jargon).  

As a final source of information, I extract features from CrossLead's database related to the location of the objective within the company structure as well as the skill level (measured by their skill assessment survey) of the person responsible for the objective. 

In order to combine information from the bag of words model and the extra features that I extracted from the data, I use a version of [stacked generalization](http://machine-learning.martinsewell.com/ensembles/stacking/), a method that allows for combining predictions from different sources in order to improve accuracy. Thus my new model will include as a feature the predictions (on the whole dataset) from the BoW model, as well as the syntactic and skill features that I just mentioned. I train (and test) in the same subset of the data as the previous model a K-Neighbors classifier. This increases my accuracy significantly, classifying correctly 76% of the test set after performing a 10-fold cross validated grid search on the parameters.

# <a name="accuracy">5. How accurate is it?</a>

If we look at the confusion matrix below, we can see that the model performs best when finding objectives that are either `forgotten` or `zero`. The worst performance is in the class `more than half`, in which around 40% of the samples are being classified incorrectly. This is partly due to the class imbalance problem (the fourth class is around 12% of the data), and will likely correct itself as CrossLead's platform grows (they are a relatively new company) and data for that class becomes more abundant.

<p align="center">
  <img src="https://www.dropbox.com/s/1knse409mqqtqoo/Screenshot%202017-02-09%2017.30.28.png?raw=1" />
</p>

Given that CrossLead is a relatively new company and they don't have a lot of data, it may be good to see if the model will perform better over time. To do this I compared the learning rates of two algorithms, as well as the dependance on a feature that I created and added to the model to account for time. 

The plot below shows the learning rate of a K-neighbors classifier and of a decision tree as we vary the sample size. The decision tree classifier (without the time feature) statistically outperforms K-Neigbors, although K-Neighbors grows faster. However, the correlation between time and *objectives* will likely fade away when CrossLead has a larger base of objective and a single company is unlikely to change the composition of the data very much, so it is likely that decision trees will do better as the database grows.  

<p align='center'>
  <img src="https://www.dropbox.com/s/uenkpxu1jym3h7p/Screenshot%202017-02-13%2000.34.18.png?raw=1" />
</p>

# <a name="how_write">6. So, how should we write?</a>

The model gives some very intuitive predictions. For example, a fictitious objective from 'X-Inc' such as:

> Discuss X Inc's project capacity and plans

Would never be updated (the model would predict the label `forgotten`), but suppose that 'X-Inc decided to frame it in a more concrete way, after all, discussing something is never a final end in itself. Suppose then they changed only one word:

> Connect X Inc's project capacity and plans. 

This would be given the label `zero`, the objective has been updated but the user has reported zero progress. If they now changed it to:

> Refine project message to meet capacities and plans

The algorithm would predict it would get progress up to 50% `half or less`. And finally, to make it fall in the `more than half` category:

> Shape functions in project to meet A Inc's capacities and plans

Not all of the predictions are as interpretable, but it's good to see a couple of sanity checks.  

# <a name="what_next">7. What's next?</a>

A predictive model of objective completion is the first step towards building an internal analytics tool within CrossLead's platform. The model will help account managers better understand their own clients use of the product. As the data grows and the model's predictions become more generalizable, it will also serve to coach clients to write their objectives in ways that make them more likely to be updated.  

Finally, a small note on causality. A predictive model like this one can be very accurate at finding the features that are most salient in a particular class --i.e. the words most associated with successful objectives. As precise as those predictions can be, the counterfactual is not guaranteed in any way. In this case, it means that changing the way you write an objective is not necessarily going to make someone more successful at achieving the goal. It may nudge you in that direction (and in this case, I don't think it will hurt), but it will probably be by making users think more thoroughly about their goals as they make an effort to write them in clearer terms. 
