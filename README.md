mlforecast
================

<!-- WARNING: THIS FILE WAS AUTOGENERATED! DO NOT EDIT! -->

[![CI](https://github.com/Nixtla/mlforecast/actions/workflows/ci.yaml/badge.svg)](https://github.com/Nixtla/mlforecast/actions/workflows/ci.yaml)
[![Python](https://img.shields.io/pypi/pyversions/mlforecast.png)](https://pypi.org/project/mlforecast/)
[![PyPi](https://img.shields.io/pypi/v/mlforecast?color=blue.png)](https://pypi.org/project/mlforecast/)
[![conda-forge](https://img.shields.io/conda/vn/conda-forge/mlforecast?color=blue.png)](https://anaconda.org/conda-forge/mlforecast)
[![License](https://img.shields.io/github/license/Nixtla/mlforecast.png)](https://github.com/Nixtla/mlforecast/blob/main/LICENSE)

## Install

### PyPI

`pip install mlforecast`

If you want to perform distributed training, you can instead use
`pip install "mlforecast[distributed]"`, which will also install
[dask](https://dask.org/). Note that you’ll also need to install either
[LightGBM](https://github.com/microsoft/LightGBM/tree/master/python-package)
or
[XGBoost](https://xgboost.readthedocs.io/en/latest/install.html#python).

### conda-forge

`conda install -c conda-forge mlforecast`

Note that this installation comes with the required dependencies for the
local interface. If you want to perform distributed training, you must
install dask (`conda install -c conda-forge dask`) and either
[LightGBM](https://github.com/microsoft/LightGBM/tree/master/python-package)
or
[XGBoost](https://xgboost.readthedocs.io/en/latest/install.html#python).

## How to use

The following provides a very basic overview, for a more detailed
description see the
[documentation](https://nixtla.github.io/mlforecast/).

### Data setup

Store your time series in a pandas dataframe in long format, that is,
each row represents an observation for a specific serie and timestamp.

``` python
from mlforecast.utils import generate_daily_series

series = generate_daily_series(
    n_series=20,
    max_length=100,
    n_static_features=1,
    static_as_categorical=False,
    with_trend=True
)
series.head()
```

<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>ds</th>
      <th>y</th>
      <th>static_0</th>
    </tr>
    <tr>
      <th>unique_id</th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>id_00</th>
      <td>2000-01-01</td>
      <td>1.751917</td>
      <td>72</td>
    </tr>
    <tr>
      <th>id_00</th>
      <td>2000-01-02</td>
      <td>9.196715</td>
      <td>72</td>
    </tr>
    <tr>
      <th>id_00</th>
      <td>2000-01-03</td>
      <td>18.577788</td>
      <td>72</td>
    </tr>
    <tr>
      <th>id_00</th>
      <td>2000-01-04</td>
      <td>24.520646</td>
      <td>72</td>
    </tr>
    <tr>
      <th>id_00</th>
      <td>2000-01-05</td>
      <td>33.418028</td>
      <td>72</td>
    </tr>
  </tbody>
</table>
</div>

### Models

Next define your models. If you want to use the local interface this can
be any regressor that follows the scikit-learn API. For distributed
training there are `LGBMForecast` and `XGBForecast`.

``` python
import lightgbm as lgb
import xgboost as xgb
from sklearn.ensemble import RandomForestRegressor

models = [
    lgb.LGBMRegressor(),
    xgb.XGBRegressor(),
    RandomForestRegressor(random_state=0),
]
```

### Forecast object

Now instantiate a `MLForecast` object with the models and the features
that you want to use. The features can be lags, transformations on the
lags and date features. The lag transformations are defined as
[numba](http://numba.pydata.org/) *jitted* functions that transform an
array, if they have additional arguments you can either supply a tuple
(`transform_func`, `arg1`, `arg2`, …) or define new functions fixing the
arguments. You can also define differences to apply to the series before
fitting that will be restored when predicting.

``` python
from mlforecast import MLForecast
from numba import njit
from window_ops.expanding import expanding_mean
from window_ops.rolling import rolling_mean


@njit
def rolling_mean_28(x):
    return rolling_mean(x, window_size=28)


fcst = MLForecast(
    models=models,
    freq='D',
    lags=[7, 14],
    lag_transforms={
        1: [expanding_mean],
        7: [rolling_mean_28]
    },
    date_features=['dayofweek'],
    differences=[1],
)
```

### Training

To compute the features and train the models call `fit` on your
`Forecast` object. Here you have to specify the columns that:

- Identify each serie (`id_col`). If the series identifier is the index
  you can specify `id_col='index'`
- Contain the timestamps (`time_col`). Can also be integers if your data
  doesn’t have timestamps.
- Are the series values (`target_col`)
- Are static (`static_features`). These are features that don’t change
  over time and can be repeated when predicting.

``` python
fcst.fit(series, id_col='index', time_col='ds', target_col='y', static_features=['static_0'])
```

    MLForecast(models=[LGBMRegressor, XGBRegressor, RandomForestRegressor], freq=<Day>, lag_features=['lag-7', 'lag-14', 'expanding_mean_lag-1', 'rolling_mean_28_lag-7'], date_features=['dayofweek'], num_threads=1)

### Predicting

To get the forecasts for the next `n` days call `predict(n)` on the
forecast object. This will automatically handle the updates required by
the features using a recursive strategy.

``` python
predictions = fcst.predict(14)
predictions
```

<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>ds</th>
      <th>LGBMRegressor</th>
      <th>XGBRegressor</th>
      <th>RandomForestRegressor</th>
    </tr>
    <tr>
      <th>unique_id</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>id_00</th>
      <td>2000-04-04</td>
      <td>69.082830</td>
      <td>67.761337</td>
      <td>68.184016</td>
    </tr>
    <tr>
      <th>id_00</th>
      <td>2000-04-05</td>
      <td>75.706024</td>
      <td>74.588699</td>
      <td>75.470680</td>
    </tr>
    <tr>
      <th>id_00</th>
      <td>2000-04-06</td>
      <td>82.222473</td>
      <td>81.058289</td>
      <td>82.846249</td>
    </tr>
    <tr>
      <th>id_00</th>
      <td>2000-04-07</td>
      <td>89.577638</td>
      <td>88.735947</td>
      <td>90.201271</td>
    </tr>
    <tr>
      <th>id_00</th>
      <td>2000-04-08</td>
      <td>44.149095</td>
      <td>44.981384</td>
      <td>46.096322</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>id_19</th>
      <td>2000-03-23</td>
      <td>30.236012</td>
      <td>31.949095</td>
      <td>32.656369</td>
    </tr>
    <tr>
      <th>id_19</th>
      <td>2000-03-24</td>
      <td>31.308269</td>
      <td>32.765919</td>
      <td>33.624488</td>
    </tr>
    <tr>
      <th>id_19</th>
      <td>2000-03-25</td>
      <td>32.788550</td>
      <td>33.628864</td>
      <td>34.581486</td>
    </tr>
    <tr>
      <th>id_19</th>
      <td>2000-03-26</td>
      <td>34.086976</td>
      <td>34.508457</td>
      <td>35.553173</td>
    </tr>
    <tr>
      <th>id_19</th>
      <td>2000-03-27</td>
      <td>34.288968</td>
      <td>35.411613</td>
      <td>36.526505</td>
    </tr>
  </tbody>
</table>
<p>280 rows × 4 columns</p>
</div>

### Visualize results

``` python
import matplotlib.pyplot as plt
import pandas as pd

fig, ax = plt.subplots(nrows=2, ncols=2, figsize=(12, 6), gridspec_kw=dict(hspace=0.3))
for i, (cat, axi) in enumerate(zip(series.index.categories, ax.flat)):
    pd.concat([series.loc[cat, ['ds', 'y']], predictions.loc[cat]]).set_index('ds').plot(ax=axi)
    axi.set(title=cat, xlabel=None)
    if i % 2 == 0:
        axi.legend().remove()
    else:
        axi.legend(bbox_to_anchor=(1.01, 1.0))
fig.savefig('figs/index.png', bbox_inches='tight')
plt.close()
```

![](https://raw.githubusercontent.com/Nixtla/mlforecast/main/figs/index.png)

## Sample notebooks

- [m5](https://www.kaggle.com/code/lemuz90/m5-mlforecast-eval)
- [m4](https://www.kaggle.com/code/lemuz90/m4-competition)
- [m4-cv](https://www.kaggle.com/code/lemuz90/m4-competition-cv)
