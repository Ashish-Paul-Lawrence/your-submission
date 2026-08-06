A1

Corpus description
Corpus: FLORES-200 Evaluation Dataset (English, Hindi, Malayalam, Tamil)
Languages used:
English (eng_Latn) 
Hindi (hin_Deva) 
Malayalam (mal_Mlym) 
Tamil (tam_Taml) 
Corpus size
dev: 997 lines
devtest: 1012 lines
Total: 2009 lines

Domain
The FLORES-200 corpus is a multilingual evaluation corpus designed for machine translation research. It contains short, carefully translated sentences spanning a variety of general-purpose topics, including:
daily life 
education 
health 
news 
culture 
science 
travel 
society 
The corpus is balanced across languages, with each sentence translated professionally so that every language contains the same semantic content.

Preprocessing
The extracted corpus required minimal preprocessing.
The dataset was downloaded using the Hugging Face datasets library. 
Only four language subsets (English, Hindi, Malayalam, and Tamil) were extracted. 
The dev and devtest splits were saved as UTF-8 encoded .txt files. 
Each line in a file corresponds to one sentence. 
Sentence alignment was preserved across all languages; for example, line n in every language file represents the same source sentence translated into that language. 
No additional tokenization, stemming, lowercasing, or text normalization was performed. 

Corpus limitations (sample size and domain caveats)
The FLORES corpus is intended for evaluation rather than training, so its size is relatively small, containing only 2,009 sentences per language. While it covers a broad range of general topics, it does not represent specialized domains such as medicine, law, finance, or social media conversations. The sentences are professionally translated and well-formed, meaning they do not reflect spelling mistakes, colloquial language, dialectal variations, or code-switching commonly found in real-world text. Consequently, analyses or models based solely on this corpus may not generalize well to informal or domain-specific language. These limitations are expected because the dataset was designed to provide a standardized benchmark for multilingual evaluation rather than a comprehensive representation of natural language use.
