---
jupyter:
  jupytext:
    formats: ipynb,md
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.14.5
  kernelspec:
    display_name: Python 3 (ipykernel)
    language: python
    name: python3
---

```python
import pandas
import pickle
import os
#import numpy as np

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
df2 = df2.rename(columns={"CD_AGEGROUP": "AGE", "MS_NUM_DEATH": "DEATH"})
df2['YEAR'] = df2['DT_DATE'].transform(lambda x: x.year)
df2['MONTH'] = df2['DT_DATE'].transform(lambda x: x.month)
for col in ["CD_ARR", "CD_PROV", "CD_REGIO", "CD_SEX", "NR_WEEK", "NR_YEAR", "DT_DATE"]:
    del df2[col]

# This contains Cause-of-death, but only until 2018 :-/
#url = "https://statbel.fgov.be/sites/default/files/files/opendata/COD/opendata_COD_cause.xlsx"
#df2 = read_cached("opendata_COD_cause.pickle", url, pandas.read_excel)
#df2 = df2.rename(columns={"CD_AGE": "AGE", "COUNT": "DEATH"})
#for col in ["CD_RGN_REFNIS", "MS_SEX_", "CHAP_COD"]:
#    del df2[col]

total_death_df = pandas.concat([df1, df2]).groupby(['YEAR', 'AGE'], as_index=False).aggregate({'DEATH': 'sum'})
```

```python
url = "https://statbel.fgov.be/sites/default/files/files/documents/bevolking/5.1%20Structuur%20van%20de%20bevolking/Population_par_commune.xlsx"
data = {}
for year in range(2000, 2023):
    df = read_cached(f"population_par_commune_{year}.pickle", url, lambda url: pandas.read_excel(url, f"Population {year}"))
    data[year] = df[df['Unnamed: 1'] == 'Belgique'].loc[3][4]
```

```python
import numpy as np
import re

#df = pandas.read_csv("/Users/gabriel/Downloads/export.csv")
df = df[pandas.isna(df['Région'])]
del df['Région']
del df['Belgique']

df = df.rename(columns={f"Population au 01 janvier {year}": str(year) for year in range(2000, 2022)})
df = df.rename(columns={"Classe d’âges": "AGE"})
i = lambda s: int(re.search(r'\d+', s).group())
df['AGE'] = df['AGE'].transform(
    lambda x: ['0-24','25-44','45-64','65-74','75-84','85+'][np.argwhere(i(x) >= np.array([0,25,45,65,75,85,200]))[-1,0]]
)
total_pop_df = df.melt(id_vars=["AGE"], var_name="YEAR", value_name="TOTAL").groupby(['YEAR', 'AGE'], as_index=False).aggregate({'TOTAL':'sum'})
```

```python
url = "https://epistat.sciensano.be/Data/COVID19BE_MORT.csv"
df = read_cached("COVID19BE_MORT.pickle", url, pandas.read_csv)
for col in ["REGION", "SEX"]:
    del df[col]
df = df.rename(columns={"DATE": "MONTH", "AGEGROUP": "AGE", "DEATHS": "COVID"}) 
df["YEAR"] = df["MONTH"].transform(lambda x: int(x[0:4]))
df['MONTH'] = df['MONTH'].transform(lambda x: int(x[5:7]))
covid_death_df = df.groupby(['YEAR', 'AGE'], as_index=False).aggregate({'COVID': 'sum'})
```

```python
from matplotlib import pyplot as plt
from functools import reduce

dfs = [covid_death_df, total_death_df, total_pop_df]
k = 'YEAR'
for df in dfs:
    df[k] = df[k].astype(int)
data = reduce(lambda a,b: pandas.merge(a,b, on=['YEAR', 'AGE'], how='outer'), dfs).sort_values('YEAR')
data = data[data['YEAR']<2022]
ages = set(data['AGE'])
data.fillna(0, inplace=True)

df = data.copy()
df['OTHER'] = (df['DEATH']-df['COVID'])/data['TOTAL']
df['COVID'] = df['COVID']/df['TOTAL']
del df['DEATH']
del df['TOTAL']
fig, axes = plt.subplots(len(ages)+1, 1, figsize=(10,30))
fig.suptitle("Belgian death ratio per age group", y=0.89)
for age, ax in zip(sorted(ages), axes):
    y = df[(df["YEAR"]==2020) & (df['AGE']==age)][['COVID','OTHER']].values.sum()
    ax.text(20,y, "level of\n2020", color='gray', horizontalalignment='center')
    ax.plot([-1,22],[y,y], color='gray', zorder=1)
    ax.set_ylabel(age)
    df[df['AGE']==age].plot(kind='bar', y=['OTHER', 'COVID'], x='YEAR', ax=ax, stacked=True, zorder=2)
    ax.set_xlabel("")
    ax.legend(loc='upper center')


ax = axes[-1]
df = data.groupby(['YEAR'], as_index=False).aggregate({'COVID':'sum', 'DEATH':'sum', 'TOTAL':'sum'})
df['OTHER'] = (df['DEATH']-df['COVID'])/df['TOTAL']
df['COVID'] = df['COVID']/df['TOTAL']
del df['DEATH']
del df['TOTAL']
df.plot(kind='bar', y=['OTHER', 'COVID'], x='YEAR', ax=ax, stacked=True, zorder=2)
ax.set_ylabel("Total Belgian population")
ax.set_xlabel("")
ax.legend(loc='upper center')
y = df[df["YEAR"]==2020][['COVID','OTHER']].values.sum()
ax.text(20,y, "level of\n2020", color='gray', horizontalalignment='center')
ax.plot([-1,22],[y,y], color='gray', zorder=1)

fig.savefig("death_age_group.png")
```

```python

```

```python

```
