# Introduction

**Submitted by Merck & Co., Inc.** 

**To see the full Kaggle submission and notebook, click [here](https://www.kaggle.com/weilinmeng/human-ai-co-op-toward-a-useful-vaccines-summary/)**

The [CORD-19 Research Database](https://www.kaggle.com/allen-institute-for-ai/CORD-19-research-challenge) is a growing collection of 50k+ scientific papers relating to information on variants of coronavirus. This enormous dataset poses challenges in terms of finding the right kind of information quickly for the purpose of finding useful and actionable information on today's COVID-19 pandemic. 

Here we focus efforts on the "Vaccines" task due to our specific interests in helping to find the best treatment possible and due to this task being particularly difficult compared to most others because of the need for domain expertise and knowledge. However, we note that our approach is applicable to all tasks.

The data science community has gathered together in order to build tools and technologies for the sole purpose of searching, ranking, extracting and aggregating results from this database. However, we found that the extraction + aggregation step of this challenge poses the greatest difficulty. 

We noticed that even using state of the art NLP technology can only bring us partially to the solution. Therefore, our approach focuses on leveraging existing knowledge + NLP to get us 80% of the way there, and then utilizing human review to help with the final step of aggregating and summarizing the most useful information. Our philosophy is that using human review can get us to the last 20% of the process.

**NOTE: It is highly recommended to use a GPU for this notebook to speed up some of the BERT related processes**

## Specific Technical Challenges
We summarize below the specific challenges we personally encountered and noticed among other submissions.
1. Search and rank of information is often limited to titles and information extraction is limited to abstract due to scalability issues
2. The BM25 ranking algorithm is a popular way to quickly search full text, but is heavily dependent on using the correct keywords, terms and the full body text being of good quality (when often it is not)
3. Deeper context from the full text is usually not available, often lacking the necessary supplementary information to understand the extracted information fully
4. Significant human review often not leveraged, resulting in systematic biases that may not be corrected or noticed. Additionally, question Answering or BERT based summarization is not feasible due to lack of corpus specific training and/or very slow inference times. Older non-deep learning methods do not output great results either.
5. Relationships between articles are often not considered, which could be an indicator of the quality and strength of the article results (especially if the article was heavily cited).

## Our Contribution Toward Resolving the Technical Challenges
1. Since titles can be misleading, we focus our search and rank on exclusively the abstract. We accomplish this by embedding each sentence of the abstract and doing a semantic similarity search via a pretrained BioBERT deep learning model on these sentences.
2. Like many others, we also use the BM25 algorithm to search the full text. However, we use the results of the previous step (similarity search on abstract sentences) to guide our BM25 query. We take the non-stopword tokens from the similarity results and feed them into the BM25 algorithm to search for the relevant pieces of full body text in order to provide deeper context. We also remove sentences from the full body that are less than 6 words, as they are likely to contain labelings that were accidentally extracted from the PDFs. We believe that using the same language from the abstract leads to more relevant BM25 results. 
3. We provide deeper context to our results by returning multiple results from the full body instead of just outputting the top result. This is achieved by taking the top 1.5 standard deviations of BM25 scores relative to the average score. Additionally, we identify mentioned antiviral agents or related terms and associate them with each literature to provide context on which antiviral agents are being studied. Finally, we create a **knowledge graph**, the first version of which identifies quickly papers that make references to others. In this way, we can see how many times a specific paper was cited in others.
4. We then do a human manual review of our query results to carefully aggregate and summarize the most useful information pertaining to the question. We believe this is the most necessary step that current approaches cannot do automatically. 
5. We create a knowledge graph that calculates the relationships between articles. Here we contribute to a first implementation on looking at which articles cite each other.

## Approach
Here, we propose a searching and ranking framework using a combination of BioBERT sentence embeddings and BM25 ranking algorithm to guide reviewers quickly to the relevant pieces of information relative to a particular query. We specifically use a **two-stage process** in order to do 1) an initial search on sentence embeddings on abstract content, and 2) subsequently perform BM25 search of the full text based on the copied language from the first stage results.

Specifically:
0. An initial exhaustive keyword search is done to filter on the 50k+ articles pertaining to vaccines and vaccine development based on prior knowledge.
1. User provides a query related to the question of interest (i.e. "Antiviral effects of Covid").
2. Sentence embeddings are generated for the user query via pre-trained BioBERT for sentence similarity.
3. All abstracts are then split into individual sentences and embedded via the same pre-trained BioBERT for sentence similarity.
4. Using cosine distances, the embedded query is then compared against the corpus of abstract sentences to find the most similar sentence.
5. All results above a pre-specified similarity threshold (0.65 is the default) are returned (**These are the stage-1 results**). If a known keyword is included in the query, an additional filtering step is done to gaurantee that these results contain that keyword.
6. The **stage 1** results are then word tokenized, processed, and then fed into the fast Okapi BM25 ranking algorithm to search for the relevant sections of the full text that may give deeper context to the initial results (**These are the stage-2 results**). We also include metadata on the number of times the found article was cited to give context on the importance of the article
7. The output is either printed or saved, and human review is performed on the results to pick and extract out the most relevant pieces of text for final aggregation and summarization.


## Future Direction
### Knowledge Graphs
*Why a knowledge graph* - One of the initial tools in our kit we wanted to explore was instantiating a knowledge graph to represent the myriad information encoded in the corpus of text. In our early review of previous submissions we noted that many approaches were treating articles as independent objects of information. Due to the fact that science is an iterative and progressive process that builds upon historical and recent works - we wanted to capture that context and apply it in our analysis of the corpus. An initial knowledge graph was explored and built representing a variety of different entities in the corpus (authors, articles, citations, text). However, due to technical and time constraints we opted for a 100% in-memory option using networkx on a subset on entities and articles to accomplish some representation of citations.

*Technical challenges* - Due to a lack of memory efficient graph storage methods in kaggle, a lot of time was spent exploring options that would work around this constraint. Almost a full library of code was written to encode and store a knowledge graph in sqlite. While progress was made and certain entities robustly represented and encoded, it was evident that any robust graph algorithm would not run acceptably on such a data structure so the approach was shelved. Any team considering such an option must think hard about recursive querying and efficiency in such a structure to achieve something workable.

*Future work* - Here are some of the things we're hoping a knowledge graph can help us explore in the future alongside other experiments to better answer the task at hand:

- Does this work belong to a series of works?
- Can we find the articles that represent the "supporting knowledge" for a given article?
- Can we determine unique work given the context of all other works?
- Is there a network of authors contributing to the same domain?
- Can we attribute scientific rigor to certain articles, authors and apply that "trust" in the final aggregation step of the solution?
- Can we represent articles with their body text and identify similar content in other articles to help find parallel / adjacent / orthogonal work?
- Can we represent articles by the language used to cite them?

### Question Answering on Full Body Text
Question Answering is often not scalable due to the slow inference terms and the sheer number of text available in the full body. However, given if the BM25 searching algorithm can roughly find related content relative to the query, we can in the future go a step further and generate questions based on the query to extract the exact piece of information.

### Summarization for Auto Report Generation
Often times multiple pieces of text in the results are outputted based on competing similarity scores. The manual review will need to compare the multiple outputs and choose which ones that actually pertain to answering the question well. In the future, we could use a series of deep learning summarization or sentence embedding models to summarize all the results and cluster them based on their similarity. In this way, we can lesson the reading time for reviewers, and even work toward auto-report generation capabilities if paired with question answering capabilities.


# Highlighted Results
Here, we present our highlighted findings. These highlights were found by using our approach to query on multiple questions pertaining to many aspects of the task. This gets us 80% to the answer. Then, we finish the last 20% by doing a careful human review of the results and compiling them in a summary.

## TASK: Effectiveness of drugs being developed and tried to treat COVID-19 patients
**Clinical/Observational Studies**
* Lopinavir and Ritonavir seem to be common antiviral agents against COVID-19. Studies often compare new drug effectiveness against Lopinavir and Ritonavir
* Favipriavir was shown to be more effective than Lopinavir/Ritonavir control arms. In a separate study, it was also shown to be more effective than arbidol control arm
* Hydroxychloroquine was shown to be effective for recoverying of pneumonia effects of COVID-19
* Azithromycin added to Hydroxychloroquine was shown to be significantly more efficient for virus elimination, possibly because azithromycin was shown to have similar effects as hydroxychloroquine 
* Danoprevir boosted by ritonavir was shown to be safe and well tolerated in all patients
* Early and short doses of a corticosteroid called methylprednisolone was shown to be effective in treatment COVID-19

**Notes based on Review Articles**
* There have been a number of reports stating that non-steroidal anti-inflammatory drugs (NSAIDs) and corticosteroids may exacerbate symptoms in COVID-19 patients. Proper use of low-dose corticosteroids may bring survival advantages for critically ill patients, but this treatment should be strictly performed.
* Although SARS-CoV-2 replication is not entirely suppressed by interferons, viral titers are decreased by several orders of magnitude. It may be useful in the early stages of infection

## TASK: Capabilities to discover a therapeutic (not vaccine) for the disease, and clinical effectiveness studies to discover therapeutics, to include antiviral agents.
**In Vitro Studies**
* Nelfinavir acts as an HIV Protease Inibitor
* Azithromycin and Ciprofloxacin have chloroquine effects and may act as alternatives to hydroxychloroquine/chloroquine
* Sofosbuvir, Tenofovir, and Alovudine are polymerases that block Sars-Cov-2 incorporation via RdRp
* Tenofovir and Emtricitabine terminates SARS-CoV-2 RdRp catalyzed reaction and can act as preventative treatments (PreP)
* Terfiflunomide and Leflunomide were shown to have solid antiviral reduction compared to favipiravir, a drug that is already undegoing clinical trials
* Darunavir was shown to have no activity against SARS-COV-2 during In Vitro studies

**Simulations and Modeling**
* Atazanavir, Efavirenz, Dolutegravir, and Saquinavir were shown to be potential candidates of treating COVID-19 based on simulations and modeling

**Notes based on Review Articles**
* Niclosamide was able to inhibit SARS-CoV replication and totally abolished viral antigen synthesis at a concentration of 1.56 μM
* Tocilizumab is a blocker of IL-6R, which can effectively block IL-6 signal transduction pathway

## TASK: Assays to evaluate vaccine immune response and process development for vaccines, alongside suitable animal models
In querying the CORD-19 corpus for diagnostic assays to evaluate immune responses to COVID-19, it is apparent that there are several assays that are currently being developed or ameliorated. Among these are the gold standards such as ELISA and PCR which have been leveraged both alone, and as per the results below, in combination with other antibody testing methodologies such as IgG/IgM to gain more confidence in CoV-2 infection diagnosis. Among the results that were returned based on the query, there were few assays that seem to be novel in their nature, such as sereological colirometric assays and fluorescence immunochromatographic tests. 

The results summarized below were manually curated from over 100 hits for the search query used. The curation mainly took into consideration what virus the assay was addressing, as several hits were referencing viruses not in the CoV family. Additional refinement of these results will include queries to better assess the suitable animal models as well as expand to gain insight into assays from a process development standpoint as opposed to simply a diagnostic one. 
### ELISA
* A newly-developed ELISA assay for IgM and IgG antibodies against N protein of SARS-CoV-2 were used to screen the serums of admitted hospital patients with confirmed or suspected SARS-CoV-2 infection. Of the 238 patients, 194 (81.5%) were detected to be antibody (IgM and/or IgG) positive, which was significantly higher than the positive rate of viral RNA (64.3%). There was no difference in the positive rate of antibody between the confirmed patients (83.0%, 127/153) and the suspected patients (78.8%, 67/85) whose nucleic acid tests were negative.

### IgG/IgM combined test
* The sensitivity and specificity of this ease-of-use IgG/IgM combined test kit were adequate, plus short turnaround time, no specific requirements for additional equipment or skilled technicians, all of these collectively contributed to its competence for mass testing. At the current stage, it cannot take the place of SARA-CoV-2 nucleic acid RT-PCR, but can be served as a complementary option for RT-PCR. The combination of RT-PCR and IgG-IgM combined test kit could provide further insight into SARS-CoV-2 infection diagnosis.

### Serological assay
* Because most patients have rising antibody titres 10 days after symptom onset, collection of serial serum samples in the convalescent phase would be more useful. Serum IgG amounts can rise at the same time or earlier than those of IgM against SARS-CoV-2. Posterior oropharyngeal saliva samples are a non-invasive specimen more acceptable to patients and health-care workers. Unlike severe acute respiratory syndrome, patients with COVID-19 had the highest viral load near presentation, which could account for the fast-spreading nature of this epidemic. This finding emphasises the importance of stringent infection control and early use of potent antiviral agents, alone or in combination, for high-risk individuals. Serological assay can complement RT-qPCR for diagnosis.

### Antibodies assays
* Combined use of antibodies assay and qRT-PCR at the same time was able to improve the sensitivities of pathogenic-diagnosis, especially for the throat swabs group at the later stage of illness. Moreover, most of these cases with undetectable viral RNA in throat swabs specimens at the early stage of illness were able to be IgM/IgG seropositive after 7 days.

### Gold immunochromatography assay
* The colloidal gold immunochromatography assay (GICA) is a rapid diagnostic tool for novel coronavirus disease 2019 (COVID-19) infections. However, with significant numbers of false negatives, improvements to GICA are needed.

### Reverse transcription loop-mediated isothermal amplification (RT-LAMP) assay
* This assay detected SARS-CoV-2 in the mean (±SD) time of 26.28 ± 4.48 min and the results can be identified with visual observation. 

### dPCR assays
* dPCR could be a confirmatory method for suspected patients diagnosed by RT-qPCR. Furthermore, dPCR is more sensitive and suitable for low virus load specimens from the both patients under isolation and those under observation who may not be exhibiting clinical symptoms. 
* Another study showed the overall accuracy of dPCR for clinical detection was 96.3%. dPCR was shown to be powerful in detecting asymptomatic patients and suspected patients. Digital PCR is capable of checking the negative results caused by insufficient sample loading by quantifying internal reference gene from human RNA in the PCR reactions. Multi-channel fluorescence dPCR system (FAM/HEX/CY5/ROX) is able to detect more target genes in a single multiplex assay, providing quantitative count of viral load in specimens, which is a powerful tool for monitoring COVID-19 treatment.

### Novel luciferase immunosorbent assays (LISA)
* The S1-, RBD-, and NP-LISAs were more sensitive than the NTD- and S2-LISAs for the detection of anti-MERS-CoV IgG. These LISAs proved their applicability and reliability for detecting anti-MERS-CoV IgG in samples from camels, monkeys, and mice, among which the RBD-LISA exhibited excellent performance."


### Rapid serological colorimetric test
* Rapid serological test showed a sensitivity of 30% and a specificity of 89% with respect to the standard assay but, interestingly, these performances improve after 8 days of symptoms appearance. After 10 days of symptoms the predictive value of rapid serological test is higher than that of standard assay. It may detect previous exposure to the virus in currently healthy persons.

### Fluorescence immunochromatographic assay
* Fluorescence immunochromatographic assay experiments were done for detecting nucleocapsid protein of SARS-CoV-2 in nasopharyngeal swab samples and urine within 10 minutes, and evaluated its significance in diagnosis of COVID-19. We measured nucleocapsid protein in nasopharyngeal swab samples in parallel with the nucleic acid test. 100% of nucleocapsid protein positive and negative participants accord with nucleic acid test for same samples.

### RT-qPCR
* Using flu and RSV clinical specimens,researchers have collected evidence that the RT-qPCR assay can be performed directly on patient sample material from a nasal swab immersed in virus transport medium (VTM) without an RNA extraction step. This approach was used to test for the direct detection of SARS-CoV-2 reference materials spiked in VTM. The data, while preliminary, suggest that using a few microliters of these untreated samples still can lead to sensitive test results. If RNA extraction steps can be omitted without significantly affecting clinical sensitivity, the turn-around time of COVID-19 tests and the backlog we currently experience can be reduced drastically.

### Novel in vivo cell-based assay
* Reseachers developed a novel in vivo cell-based assay for examining this interaction between the N-protein and packaging signal RNA for SARS-CoV, as well as other viruses within the coronaviridae family. The N-protein specifically recognizes the SARS-CoV packaging signal with greater affinity compared to signals from other coronaviruses or non-coronavirus species. These results describe, for the first time, in vivo evidence for an interaction between the SARS-CoV N-protein and its packaging signal RNA, and demonstrate the feasibility of using this cell-based assay to further probe viral RNA-protein interactions in future studies.

## TASK: Efforts to develop prophylaxis clinical studies and prioritize in healthcare workers

Prophylaxis describes the efforts and measures to prevent infections and diseases. We summarize some of our findings on these efforts to develop prophylaxis clinical studies by categorizing them into general findings, findings related to α1-AR antagonists, and findings related to risk varied by prognostic factors.

### General findings

* Risk-adapted treatment strategy may be a useful tool for the treatment of COVID-19 patients. This strategy is associated with significant clinical manifestations alleviation and clinical imaging recovery.
* Harmonization of clinically heterogeneous endpoints within and between trials can lead to faster decision making and better management of COVID-19. 
* Early detection of elevations in serum CRP, combined with a clinical COVID-19 symptom presentation may be used as a surrogate marker for presence and severity of disease.
* There are multiple parameters of the clinical course and management of the COVID-19 that need optimization. A hindrance to this development is the vast amount of misinformation present due to scarcely sourced manuscript preprints and social media.
* Emphasize evidence-based medicine to evaluate the frequency of presentation of various symptoms to create a stratification system of the most important epidemiological risk factors for COVID-19.
* Vitamin C (L-ascorbic acid) has a pleiotropic physiological role, but there is evidence supporting the protective effect of high dose intravenous vitamin C (HDIVC) during sepsis induced ARDS.
* Epigenetic control of the ACE2 gene might be a target for prevention and therapy in COVID-19. 

### α1-AR antagonists
Preliminary findings offer a rationale for studying α1-AR antagonists in the prophylaxis of patients with COVID-19 cytokine storm syndrome (CSS) and acute respiratory distress syndrome (ARDS).
* Mortality of COVID-19 seems driven by acute respiratory distress syndrome (ARDS)
* Emerging evidence suggests that a subset of COVID-19 is characterized by the development of a CSS. 
* Pre-clinical mouse data suggests that α1-AR antagonists may be a candidate for the treatment of COVID-19.
* Using the Truven Health MarketScan Research DataBase, male men who were prescribed α1-AR antagonists in the previous year had lower odds of the composite of need for invasive mechanism ventilation and mortality compared to non-users (AOR 0.80, 95% CI 0.69-0.94, p=0.008) 

### Relative Risk of COVID-19 for Patients Varies by Prognostic Factors
COVID-19 patient outcomes vary by patient characteristics and are important considerations for COVID-19 prophylaxis. Potential important factors include interleukin-6, B lymphocyte proportion, lactate, and CD8+ T cells. 
* Compared with patients without pneumonia, those with pneumonia were 15 years older and had a higher rate of hypertension, higher frequencies of having a fever and cough, and higher levels of interleukin-6, B lymphocyte proportion, and low account of CD8+ T cells. 
* Multivariate Cox regression analysis indicated that circulating interleukin-6 and lactate independently predicted COVID-19 progression, with a hazard ratio (95%CI) of 1.052 (1.000-1.107) and 1.082 (1.013-1.155), respectively. 
