# Lending Club Loan Default Prediction Model

This project implements a machine learning model to predict loan defaults using the Lending Club dataset. The model helps lenders assess the risk of potential loan defaults by analyzing various loan and borrower characteristics.

## Project Overview

The project consists of several integrated components:
1. Data preprocessing and cleaning using SQL in BigQuery, where the machine learning model is also trained and evaluated
2. Interactive Streamlit web interface that allows users to input loan information and receive default predictions (with 99.15% accuracy)
3. Automated random value generation system for testing and demonstration purposes

## Project Structure

- `bigquery_queries.sql`: Contains SQL queries for data preprocessing and ML model training in BigQuery
- `find_ranges_for_rand.py`: Generates realistic ranges for random value generation in the interface
- `user_interface.py`: Streamlit interface implementation for user interaction
- `requirements.txt`: Project dependencies and version specifications
- **Secrets Management**: Streamlit's built-in feature is used to handle BigQuery login credentials securely, eliminating the need for external files like `secrets.toml` or `streamlit_service_key.json`. This ensures that sensitive information is managed safely within the Streamlit app environment.

## Dataset

The project uses the [Lending Club dataset from Kaggle](https://www.kaggle.com/datasets/wordsforthewise/lending-club), which contains millions of loan records. The dataset includes various features such as:
- Loan amount and funding details
- Interest rates and installment information
- Borrower characteristics (income, FICO scores, etc.)
- Credit history metrics
- Payment and recovery information

## Data Pipeline

### Data Cleaning and Preprocessing
The data was uploaded to Google Cloud Storage and is accessed via BigQuery. The cleaning process was performed using BigQuery:
- Handled missing values
- Removed unnecessary features
- Standardized data types

### Feature Engineering
Key features used in the model include items like:
- Loan amount and funding details
- Interest rates
- Borrower's annual income
- Debt-to-income ratio
- Credit scores (FICO range)
- Credit history metrics
- Payment history


# Model Development

### Model Selection
- Algorithm: Gradient Boosted Decision Tree
    Why this algorithm?: A classifier for this dataset needed to be very complex to handle the MANY intricacies within an effective classifier for this application. 
- Type: Binary Classification (Default vs. Non-default)

### Training Process

1. **Train/Test Split**: The dataset is split into training and testing sets to evaluate the model's performance. The training set consists of 80% of the data, while the testing set contains the remaining 20%. This is done using the following SQL queries:

   ```sql
   -- Split the data into training and test sets
   CREATE OR REPLACE TABLE `loan_club_dataset.train_data` AS
   WITH stratified_data AS (
     SELECT *, RAND() AS random_value
     FROM `loan_club_dataset.accepted_loans_final_cleaned`
   )
   SELECT *
   FROM stratified_data
   WHERE random_value <= 0.8;  -- 80% for training

   CREATE OR REPLACE TABLE `loan_club_dataset.test_data` AS
   WITH stratified_data AS (
     SELECT *, RAND() AS random_value
     FROM `loan_club_dataset.accepted_loans_final_cleaned`
   )
   SELECT *
   FROM stratified_data
   WHERE random_value > 0.8;  -- 20% for testing
   ```

2. **Model Training**: The model is trained using the training dataset with the following SQL query. A boosted tree classifier is used, and various parameters are set to optimize performance:

   ```sql
   CREATE OR REPLACE MODEL `loan_club_dataset.boosted_tree_model`
   OPTIONS(
     model_type='BOOSTED_TREE_CLASSIFIER',
     input_label_cols=['defaulted'],
     max_iterations=100,      
     max_tree_depth=20,
     colsample_bytree=1,   -- Uses x% of features per tree (prevents feature dominance)
     l1_reg=0.1            -- higher value encourages sparsity (if overfitting, increase this)
   ) AS
   SELECT * FROM `loan_club_dataset.train_data`;
   ```

3. **Model Testing and Evaluation**: After training, the model is tested on the test dataset, and its performance is evaluated using the following SQL queries:

   ```sql
   -- Test the model on the test set
    SELECT predicted_defaulted, predicted_defaulted_probs
    FROM
        ML.PREDICT(MODEL `loan_club_dataset.boosted_tree_model`,
             (SELECT * FROM `loan_club_dataset.test_data`));


   -- Evaluate the model
   SELECT *
   FROM
     ML.EVALUATE(MODEL `loan_club_dataset.boosted_tree_model`, 
                 (SELECT * FROM `loan_club_dataset.test_data`));
   ```

### Model Performance
The final model achieved strong performance across various metrics:

| Metric | Score |
|--------|--------|
| Precision | 0.9974 |
| Recall | 0.9311 |
| Accuracy | 0.9915 |
| F1 Score | 0.9631 |
| Log Loss | 0.0297 |
| ROC AUC | 0.9978 |

These metrics indicate:
- **Precision (0.9974)**: The model is accurate in identifying non-defaulting loans, with very few false positives
- **Recall (0.9311)**: The model identifies 93.11% of actual defaulting loans
- **Accuracy (0.9915)**: The model correctly classifies 99.15% of all loans
- **F1 Score (0.9631)**: The harmonic mean of precision and recall shows overall performance
- **Log Loss (0.0297)**: Indicates confidence in predictions
- **ROC AUC (0.9978)**: Demonstrates discrimination between defaulting and non-defaulting loans


# Interactive Interface

The project includes a Streamlit-based user interface that allows users to:
- Input loan and borrower information
- Generate predictions for loan default probability
- Test the model with random values within realistic ranges


## Usage/Proof of Concept

1. Run the Streamlit interface to generate default predictions:  
   - [Click here to launch the app](https://loan-predictor-dom-schulz.streamlit.app/)
2. Input loan information through the interface or hit "Generate New Random Values" at the bottom.  
   - Note: the fields are randomly generated when the app is loaded.  
3. View the prediction results


# Future Improvements

1. Look into more specific feature importance and better select features for training. It is lucky how well this model is performing given the quick, simple cleaning process.
2. Look into performance of other models and weighting them in an ensemble.


# Challenges

1. Model training time 
    - Utilizing the whole dataset for every training iteration after making adjustments to parameters was extremely slow (ie. ~10 minutes per training, depending on certain parameters). I limited the training and testing data to smaller training sizes (about 50,000 - 100,000 rows) to speed up my first training efforts. Once I had a general idea of where to place parameters, I increased the training and testing sets back to their original size.  
2. Permissions within Google Cloud
    - There were many different permissions need for BigQuery and model access for service accounts in Google Cloud. This caused many errors for me to work through during this project. 
3. Random number generation for the streamlit app
    - Truly random number/value generation for the predictor interface naturally caused many issues with generating reasonable values. So this resulted in more time spent creating `find_ranges_for_rand.py` to get hard coded ranges and values to generate within. 
4. Technicality of Model Execution
    - Just like in my "Future Improvements" section, I believe that I skimmed over some items involving my training and testing, and could improve my model and understanding of the dataset. I believe I somewhat got lucky with this classifier working so well with having to do little testing with parameters and types of models. 