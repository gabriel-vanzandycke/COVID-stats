---
jupyter:
  jupytext:
    formats: ipynb,md
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.13.4
  kernelspec:
    display_name: Python 3 (ipykernel)
    language: python
    name: python3
---

```python
import pandas
import pickle
import os
import numpy as n

def read_cached(cache, url, function):
    if os.path.isfile(cache):
        print(f"'{cache}' found. Loading it from pickle.")
        return pickle.load(open(cache, "rb"))
    print(f"'{cache}' missing. Downloading it from url and saving a cached version in pickle.")
    print(url)
    df = function(url)
    pickle.dump(df, open(cache, "wb"))
    return df
```

```python
url = "https://statbel.fgov.be/sites/default/files/files/opendata/COD/opendata_COD_cause%202000-2008.xlsx"
df1 = read_cached("opendata_COD_cause 2000-2008.pickle", url, pandas.read_excel)
df1 = df1.rename(columns={"CD_AGE": "AGE", "COUNT": "DEATH"})
for col in ["CD_RGN_REFNIS", "MS_SEX_", "CHAP_COD"]:
    del df1[col]

url = "https://statbel.fgov.be/sites/default/files/files/opendata/deathday/DEMO_DEATH_OPEN.xlsx"
df2 = read_cached("DEMO_DEATH_OPEN.pickle", url, pandas.read_excel)
df2 = df2.rename(columns={"CD_AGEGROUP": "AGE", "NR_YEAR": "YEAR", "DT_DATE": "MONTH", "MS_NUM_DEATH": "DEATH"})
df2['MONTH'] = df2['MONTH'].transform(lambda x: x.month)
for col in ["CD_ARR", "CD_PROV", "CD_REGIO", "CD_SEX", "NR_WEEK"]:
    del df2[col]

df = pandas.concat([df1, df2])
df = df.groupby(['MONTH', 'YEAR', 'AGE'], as_index=False).aggregate({'DEATH': 'sum'})
pickle.dump(df, open("death_month_sex_age_2000_2021.pickle", "wb"))
```

```python
import re
import numpy as np
from urllib.request import Request, urlopen

# WARNING /!\ data after 2019 is probably an extrapolation and I don't trust 100% data before 2019
def read_csv(year):
    url = f"https://www.populationpyramid.net/api/pp/56/{year}/?csv=true"
    req = Request(url)
    req.add_header('User-Agent', 'Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:77.0) Gecko/20100101 Firefox/77.0')
    content = urlopen(req)
    df = pandas.read_csv(content)
    df = df.rename(columns={'Age': 'AGE'})
    df['TOT'] = df['M'] + df['F']
    del df['M']
    del df['F']
    i = lambda s: int(re.search(r'\d+', s).group())
    df['AGE'] = df['AGE'].transform(
        lambda x:['0-24','25-44','45-64','65-74','75-84','85+'][np.argmin(np.array([0,25,45,65,85,200]) < i(x))]
    )
    df = df.groupby('AGE', as_index=False).aggregate({'TOT':'sum'})
    df['YEAR'] = year
    return df
df_tot = pandas.concat([read_csv(year) for year in range(2000,2022)])
```

```python
from matplotlib import pyplot as plt

ages = set(df['AGE'])
data = pandas.merge(df.groupby(['YEAR', "AGE"], as_index=False).aggregate({'DEATH':'sum'}), df_tot)
data['RATIO'] = data['DEATH']/data['TOT']
fix, axes = plt.subplots(len(ages), 1, figsize=(10,30))
for age, ax in zip(sorted(ages), axes):
    ax.set_title(age)
    data[data['AGE']==age].plot(kind='bar', y='RATIO', x='YEAR', ax=ax, legend=False)
```

```python

```

```python

```
