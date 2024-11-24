NooraHealth Data Analysis

Overview
This project focuses on analyzing messaging data to extract insights and visualize patterns for better understanding of user engagement and message flow. The pipeline involves data extraction, transformation, validation, and visualization using Google BigQuery and Python (with libraries like Pandas and Plotly).

Table of Contents
1. Project Workflow
2. Key Features
3. Setup Instructions
4. Data Transformation Queries
5. Validation Checks
6. Visualizations
7. Technologies Used
8. Future Enhancements
9. Contact

Project Workflow
Extract and Load Data:
Import data files into BigQuery and format them with headers.
Data Transformation:
Combine and transform tables to create a unified dataset.
Identify duplicates and validate data for quality.
Data Visualization:
Generate visual insights using Python to explore trends, engagement metrics, and message flow.

Key Features
Unified Data Transformation: Combines and structures message data into a single cohesive table.
Robust Validation: Includes checks for missing values, duplicates, and invalid statuses.
Insightful Visualizations:
Trends in total and active users.
Message read rates and response times.
Status breakdown of outbound messages.

Setup Instructions
Prerequisites
Google BigQuery for data transformation.
Python 3.x installed with the following libraries:
pandas
plotly
datetime
Steps to Run:
Set up your environment and install dependencies:
Execute the SQL queries in BigQuery to transform the data.
Export the transformed data as a CSV file.
pip install pandas plotly
Use the Jupyter Notebook provided to visualize the data

Data Transformation Queries
1. Combine Tables into a Unified Dataset

CREATE TABLE `Message.Transformed` AS
SELECT 
    M.id AS message_id,
    M.message_type,
    M.masked_addressees,
    M.masked_author,
    M.content,
    M.author_type,
    M.direction,
    M.external_id,
    M.external_timestamp,
    M.masked_from_addr,
    M.is_deleted,
    M.last_status,
    M.last_status_timestamp,
    M.rendered_content,
    M.source_type,
    M.uuid,
    M.inserted_at AS message_inserted_at,
    M.updated_at AS message_updated_at,
    S.status,
    S.timestamp AS status_timestamp,
    S.inserted_at AS status_inserted_at,
    S.updated_at AS status_updated_at
FROM `Message.messageM` M
LEFT JOIN `Message.statusS` S
ON M.uuid = S.message_uuid;

2. Detect and Flag Duplicates
SELECT 
    message_id, 
    content, 
    COUNT(*) AS duplicate_count
FROM `Message.Transformed`
WHERE SAFE.PARSE_TIMESTAMP('%m/%d/%Y %H:%M:%S', message_inserted_at) IS NOT NULL
GROUP BY message_id, content, DATE(PARSE_TIMESTAMP('%m/%d/%Y %H:%M:%S', message_inserted_at))
HAVING COUNT(*) > 1;

Validation Checks
Check 1- count missing values
SELECT COUNT(*) AS missing_values_count
FROM `Message.Transformed`
WHERE message_id IS NULL
   OR content IS NULL
   OR message_inserted_at IS NULL;

Check 2- count invalid status
SELECT status, COUNT(*) AS status_count
FROM `Message.Transformed`
GROUP BY status
HAVING status NOT IN ('sent', 'delivered', 'read', 'failed', 'deleted');

Check 3- count duplicate message_ids
SELECT message_id, COUNT(*) AS duplicate_count
FROM `Message.Transformed`
GROUP BY message_id
HAVING COUNT(*) > 1;

Visualizations
1. Total and Active Users Over Time
# Step 1: Convert message_inserted_at to datetime for analysis

df['message_inserted_at'] = pd.to_datetime(df['message_inserted_at'], errors='coerce')

# Step 2: Define the date range for the last 3 months
end_date = df['message_inserted_at'].max()  # Get the most recent message date
start_date = end_date - pd.DateOffset(months=3)  # Define the starting point as 3 months back

# Step 3: Filter the dataset to include only messages from the last 3 months
df_recent = df[df['message_inserted_at'] >= start_date]

# Step 4: Calculate the total number of unique users (those who sent or received messages)
# Total users are those with unique author IDs
total_users = df_recent['masked_author'].nunique()

# Step 5: Calculate the number of active users (those who sent inbound messages)
# Active users are those who have sent inbound messages
active_users = df_recent[df_recent['direction'] == 'inbound']['masked_author'].nunique()

# Step 6: Group the data by week and calculate the number of total and active users per week
# Convert message_inserted_at to a weekly period (e.g., week starting on Monday)
df_recent['week'] = df_recent['message_inserted_at'].dt.to_period('W').astype(str)

# Group the data by 'week' and calculate unique counts for total and active users
user_counts = df_recent.groupby('week').agg(
    total_users=('masked_author', 'nunique'),
    active_users=('masked_author', lambda x: x[df_recent['direction'] == 'inbound'].nunique())
).reset_index()

# Step 7: Visualize the total and active users over time
fig = px.line(user_counts, x='week', y=['total_users', 'active_users'],
              labels={'value': 'Number of Users', 'week': 'Week'},
              title='Total and Active Users Over Time')
fig.show()

![image](https://github.com/user-attachments/assets/65ba5028-3a05-4315-ad8e-386ae79badc3)


3. Fraction of Sent Messages Read
# Step 8: Create a new column to classify message status (Read or Unread)
# The 'status' column indicates whether the message was read ('read') or not
df['read_status'] = df['status'].apply(lambda x: 'Read' if x == 'read' else 'Unread')

# Step 9: Calculate the fraction of sent messages that are read
# Filter outbound messages and calculate the fraction of 'Read' messages
sent_messages = df[df['direction'] == 'outbound']
read_fraction = sent_messages[sent_messages['read_status'] == 'Read'].shape[0] / sent_messages.shape[0]

# Step 10: Calculate the time difference between when a message was sent and when it was read
# Convert the timestamp columns to datetime objects
df['sent_timestamp'] = pd.to_datetime(df['message_inserted_at'], errors='coerce')
df['read_timestamp'] = pd.to_datetime(df['status_timestamp'], errors='coerce')

# Step 11: Calculate the time difference between sent and read timestamps in seconds
df['time_to_read'] = (df['read_timestamp'] - df['sent_timestamp']).dt.total_seconds()

# Step 12: Filter the messages that were read and calculate the average time to read
read_messages = df[df['read_status'] == 'Read']
time_to_read_mean = read_messages['time_to_read'].mean()

# Step 13: Visualize the fraction of sent messages that were read
fig = px.bar(x=['Read Fraction'], y=[read_fraction], title="Fraction of Sent Messages that are Read")
fig.show()

# Step 14: Visualize the time taken to read the messages (for those that were read)
fig = px.histogram(read_messages, x='time_to_read', nbins=20, title="Time Between Message Sent and Read")
fig.show()

![image](https://github.com/user-attachments/assets/46475341-a05b-42dd-b9bb-90975dea78da)
![image](https://github.com/user-attachments/assets/8e68a66c-b330-4100-9c91-bf4888f9a1b9)

5. Outbound Messages by Status
#Number of Outbound Messages in the Last Week by Status
# Step 15: Define the date range for the last week
end_date = df['message_inserted_at'].max()  # Get the most recent message date
start_date = end_date - pd.DateOffset(weeks=1)  # Define the starting point as 1 week back

# Step 16: Filter data for outbound messages from the last week
df_last_week = df[(df['message_inserted_at'] >= start_date) & (df['direction'] == 'outbound')]

# Step 17: Group by message status and count occurrences
message_status_counts = df_last_week['status'].value_counts().reset_index()
message_status_counts.columns = ['status', 'count']

# Step 18: Visualize the number of outbound messages by status in the last week
fig = px.bar(message_status_counts, x='status', y='count', title="Number of Outbound Messages by Status in the Last Week")
fig.show()

![image](https://github.com/user-attachments/assets/dd24411e-0c50-4552-b814-146f8bbb82de)

Technologies Used
Google BigQuery: SQL-based data transformation.
Python: Data visualization and analysis.
Plotly: Interactive visualizations.
Pandas: Data manipulation.

Future Enhancements
Automate the pipeline from BigQuery to visualization.
Add more visualizations for user engagement metrics.
Enhance validation checks for dynamic rule updates.

Contact
For questions or contributions, feel free to reach out:
Email: mrunmaymuduli711@gamail.com
