# Dataflows schema drift behavior overview 

In Synapse dataflows, schema drift enables transformation processes to adapt dynamically when changes occur in the source schema. 
If new columns are introduced or existing ones are modified at the source, the transformation logic remains robust and does not result in script failures.
However, while the dataflow job will successfully execute, it is important to understand that the altered schema will not be automatically propagated to the destination table. 
Data engineers must implement additional steps to update the destination schema accordingly, otherwise, any new or altered columns may not be captured, potentially leading to data loss. Refer to the example below for further clarification.


# Example


## Current table:

![image](https://github.com/user-attachments/assets/601490cf-4a98-4e22-a81d-6460bfa3c97d)

## Dataflow transformation with schema drift enabled: 

![image](https://github.com/user-attachments/assets/1caec044-5ca1-4a20-bf6c-eea8ef695908)

## Schema Updated e.g. Source & Sink Different: 

![image](https://github.com/user-attachments/assets/9695459e-5361-4666-918c-d1ee6f9f765a)

## Dataflows is still successful:

![image](https://github.com/user-attachments/assets/27c07b6f-95a4-4f17-bd5a-8fc8a4fcd77c)


![image](https://github.com/user-attachments/assets/7181b83e-d18a-43fa-98eb-ea27446bbb28)

## However, data is lost on the sink side as schema drift does not modify SQL DDL as there is no MembershipLevel column on the sink side: 

![image](https://github.com/user-attachments/assets/d40b9479-05d2-4df7-a858-2a6080a69745)


# Potential Solution

One potential solution involves employing a lookup activity followed by a foreach loop to iterate through each column. 
By querying the INFORMATION_SCHEMA.COLUMNS table with an “IF NOT EXISTS” condition, you can identify and add any missing columns to the destination schema without unnecessarily running the command on a production system. 
This process ensures that the schema is updated appropriately before executing your dataflow or other copy operations, thereby preventing data loss due to schema drift.

Please see below. 

![image](https://github.com/user-attachments/assets/3a2f50ae-249f-453c-a479-bb6f8959ec22)



![image](https://github.com/user-attachments/assets/8e889331-00ef-4901-98c0-61bc70175cc3)


```
SELECT COLUMN_NAME, DATA_TYPE, CHARACTER_MAXIMUM_LENGTH
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'Customers' AND TABLE_SCHEMA = 'dbo' order by COLUMN_NAME
```

![image](https://github.com/user-attachments/assets/b71cce13-8c01-4068-9bfa-53f10097417e)


![image](https://github.com/user-attachments/assets/8ac52ac6-4f00-4730-92e0-384fa1039277)



##Script task: 

![image](https://github.com/user-attachments/assets/8d88cef3-1a70-4dc9-bfab-27de7a7013f8)


```
@concat(
  'IF NOT EXISTS (',
    ' SELECT * FROM INFORMATION_SCHEMA.COLUMNS',
    ' WHERE TABLE_NAME = ''customersink1''',
    '   AND COLUMN_NAME = ''', item().COLUMN_NAME, '''',
  ') ',
  'BEGIN ',
    'ALTER TABLE customersink1 ADD ',
    item().COLUMN_NAME, ' ',
    if(
      or(
        or(
          equals(item().DATA_TYPE, 'varchar'),
          equals(item().DATA_TYPE, 'nvarchar')
        ),
        or(
          equals(item().DATA_TYPE, 'char'),
          equals(item().DATA_TYPE, 'nchar')
        )
      ),
      concat(
        item().DATA_TYPE,
        '(',
        string(item().CHARACTER_MAXIMUM_LENGTH),
        ')'
      ),
      item().DATA_TYPE
    ),
    ';',
  ' END'
)

```



***DISCLAIMER: Sample Code is provided for the purpose of illustration only and is not intended to be used in a production environment unless thorough testing has been conducted by the app and database teams. THIS SAMPLE CODE AND ANY RELATED INFORMATION ARE PROVIDED "AS IS" WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR FITNESS FOR A PARTICULAR PURPOSE. We grant You a nonexclusive, royalty-free right to use and modify the Sample Code and to reproduce and distribute the object code form of the Sample Code, provided that. You agree: (i) to not use Our name, logo, or trademarks to market Your software product in which the Sample Code is embedded; (ii) to include a valid copyright notice on Your software product in which the Sample Code is embedded; and (iii) to indemnify, hold harmless, and defend Us and Our suppliers from and against any claims or lawsuits, including attorneys fees, that arise or result from the use or distribution or use of the Sample Code.***



