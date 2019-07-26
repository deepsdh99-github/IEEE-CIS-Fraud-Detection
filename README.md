# IEEE-CIS-Fraud-Detection
Credit Card Fraud Detection

## Split data.zip into small files
split -b 50M data.zip data_split.zip

## Recreate from split files(located under data1 folder)
cat data/* > data1.zip

## Unzip files into data folder
unzip data1.zip -d data
