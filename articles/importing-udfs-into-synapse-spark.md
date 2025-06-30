# Importing user-defined functions into Synapse Spark 


Managing large PySpark functions can become cumbersome, especially when the same logic needs to be replicated across multiple scripts. 
Fortunately, there are several effective methods for storing user-defined functions, enabling seamless reuse without the need for repetitive coding. 
This documentation outlines options for organizing and reusing functions efficiently, streamlining your workflow and promoting code consistency.

# Option 1: 
One effective approach is to maintain a dedicated Spark notebook, such as a NotebookUtils, within a utils folder to house all commonly used functions. 
By including the line 
`%run /utils/NotebookUtils` 
at the beginning of any primary notebook, all necessary functions are seamlessly imported into the session each time the notebook is executed. 
This method closely mirrors the integration of a wheel file, streamlining function reuse and ensuring consistency across your environment.

![image](https://github.com/user-attachments/assets/91166031-08ac-41ff-8245-a1a4b18876aa)

![image](https://github.com/user-attachments/assets/d711c0d6-387f-40e0-bbf7-3872dd52815e)

![image](https://github.com/user-attachments/assets/452a1c6a-a80a-46c9-b38d-1caf9ddd753a)

![image](https://github.com/user-attachments/assets/f4f6913a-8e0d-4ed0-879b-191f9076b17b)

![image](https://github.com/user-attachments/assets/623e7a8a-97ff-4c89-b02c-9d4c04fd728c)

# Option 2: 

An alternative method involves utilizing a wheel file to manage your reusable functions. In this approach, you first package the necessary functions into a wheel file using the appropriate commands. Once the wheel file has been created, it is uploaded to the Synapse environment and then imported into the relevant Spark pool. This strategy mirrors the behavior of integrating external libraries, fostering consistency and reusability across your projects. However, it is important to note that this process can be more complex, often requiring up to 30 minutes to load each new file. Additionally, strict naming conventions must be followed, as each update necessitates overriding the existing wheel file to avoid conflicts and ensure that function names remain consistent when new versions of the library are deployed.

# Steps: 

Create a wheel folder. 
Establish a dedicated module directory and, within it, organize your PySpark functions as individual .py files. 
This structure not only keeps your codebase clean and maintainable but also allows for efficient importing and reuse of your functions across various scripts.

![image](https://github.com/user-attachments/assets/3edf07cc-89e3-4988-8637-a9e958897042)

```
import pandas as pd
from pyspark.sql.functions import pandas_udf
from pyspark.sql.types import StringType, IntegerType

@pandas_udf(StringType())
def alternating_case_vectorized(texts: pd.Series) -> pd.Series:
    """
    Pandas UDF to convert every other character of a string to uppercase.
    """
    results = []
    for text in texts:
        if text is None:
            results.append(None)
            continue
        result = "".join(
            ch.upper() if i % 2 == 0 else ch.lower()
            for i, ch in enumerate(text)
        )
        results.append(result)
    return pd.Series(results)

@pandas_udf(IntegerType())
def string_length_vectorized(texts: pd.Series) -> pd.Series:
    """
    Pandas UDF to calculate the length of a string.
    """
    # Note: .astype(int) not IntegerType()
    return texts.str.len().astype(int)
```

Also create a __init__ file in the same folder, it can be blank, it should look like the below: 

![image](https://github.com/user-attachments/assets/b4203d46-0714-4611-ae1f-e7125baa4583)

Go to the prior directory and create a setup.py file add your attribute values to the code.

![image](https://github.com/user-attachments/assets/0cda90ac-7b9a-44b0-8c12-9e1d1124f694)


```
import setuptools

setuptools.setup(
    name='alternatingudf2',
    version='1.0',
    author='Victor',
    author_email='victora@microsoft.com',
    description='package to convert text case',
    packages=setuptools.find_packages(),
    python_requires='>=3.4',
)
```

![image](https://github.com/user-attachments/assets/f6e594ff-bbdd-4cc5-81a6-28fffe34df80)

Navigate to the designated directory in your command shell, then execute the following command to generate the wheel file.

`Python setup.py bdist_wheel`


![image](https://github.com/user-attachments/assets/5ef9c01f-aa42-463f-b739-6aef5487fdbe)


The wheel file will be generated in the dist path:

![image](https://github.com/user-attachments/assets/fa6dd1f7-2284-4b2f-bb93-7b97e0f34d18)


Upload it into your Synapse workspace ->  Workspace packages section. 

![image](https://github.com/user-attachments/assets/3a4b8d9d-f6e9-4213-b9dd-d3d7785738ff)

Finally, upload it into your Spark Pool. Please note that this upload process can take upwards of 30 minutes. 

![image](https://github.com/user-attachments/assets/95f8a68b-ed9b-46ac-8199-d5abf79f229f)


# Performance Considerations: 
While you can use Python scalar UDFs in PySpark (for example, by registering a function with spark.udf.register("my_upper", lambda s: s.upper())), it's important to understand the limitations and consider alternatives like using pandas methods when possible.
Here’s why:
-	Serialization Overhead: Every time a row is processed by a Python UDF, data must be transferred between the JVM (Java Virtual Machine) and Python, introducing significant performance costs.
-	Lack of Optimizations: Spark treats Python UDFs as "black boxes," which means it can’t apply its built-in optimization strategies, such as Catalyst optimizations or code generation. This results in slower performance compared to native or pandas-based approaches.
-	Single-threaded Execution: Python UDFs run single-threaded per task, so you miss out on some parallelism and speed benefits that Spark’s internal processing (Tungsten engine) provides.
Using pandas functions (with pandas UDFs or built-in Spark functions), when suitable, can help bypass these bottlenecks. Pandas UDFs utilize Apache Arrow to efficiently transfer data between JVM and Python, reducing serialization overhead and enabling vectorized operations for much better performance. For many data transformation tasks, leveraging pandas or native Spark methods can make your code not only faster but also more scalable and maintainable.




***DISCLAIMER: Sample Code is provided for the purpose of illustration only and is not intended to be used in a production environment unless thorough testing has been conducted by the app and database teams. 
THIS SAMPLE CODE AND ANY RELATED INFORMATION ARE PROVIDED "AS IS" WITHOUT WARRANTY OF ANY KIND, EITHER EXPRESSED OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE IMPLIED WARRANTIES OF MERCHANTABILITY AND/OR FITNESS 
FOR A PARTICULAR PURPOSE. We grant You a nonexclusive, royalty-free right to use and modify the Sample Code and to reproduce and distribute the object code form of the Sample Code, provided that. You agree: (i) 
to not use Our name, logo, or trademarks to market Your software product in which the Sample Code is embedded; (ii) to include a valid copyright notice on Your software product in which the Sample Code is 
embedded; and (iii) to indemnify, hold harmless, and defend Us and Our suppliers from and against any claims or lawsuits, including attorneys fees, that arise or result from the use or distribution or use of the 
Sample Code.***
