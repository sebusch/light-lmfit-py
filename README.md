# Dev Notes

 This version of `lmfit` is accompanied with [DFisher_2022A](https://github.com/ADACS-Australia/dfisher_2022a) project, and the sole purpose is to make the cube fitting process faster. This version is compatible with `lmfit=1.0.3`. The faster fitting method can be called using `method="fast_leastsq"` and `fast=True` in `Model.fit`. Notice `fast_leastsq` is backed with `scipy.optimize.leastsq`, and is the only faster approach implemented at the moment.

Here I list some main changes of this customized version of `lmfit`:
1. reduce nan_policy check on residual from one per iteration to one per fitting (check `Minimizer.__residual_fast` and `Model._residual_fast`)
	- It's unlikely that Gaussian model would generate NaN value, so I think it would be enough to do nan_policy check on `residual = model - data` at the first iteration for each fitting. In the worst case, if model generates NaN value in the following iterations, the fitting will be classified as failure by scipy.optimize.leastsq
	
2. update params when the fitting is finished (check `Minimizer.__residual_fast` and `Minimizer.leastsq_fast`)
    - In `lmfit=1.0.3`, params are updated at each iteration. Here I change it to only update once when the fitting is finished. During the fitting process, the output of `scipy.optimize.leastsq` at each iteration is passed to `Model._residual_fast` as a list.

3. write a custom eval function (check [DFisher_2022A](https://github.com/ADACS-Australia/dfisher_2022a): `dfisher_2022a.models.lmfit.composite`)
   - For composite models, `lmfit=1.0.3` would evaluate the component models separately, and then combine the evaluation results together. Here I write a shortcut eval function, which evaluates the composite model at once:
   ```
   def gaussianCH(x, height=1.0, center=0.0, sigma=1.0, c=0.0):

    	return height * np.exp(-((1.0 * x - center) ** 2) / max(tiny, (2 * sigma ** 2))) + c
```


class Const_1GaussModel(lmfit.model.CompositeModel):
    def __init__(
        self, independent_vars=["x"], prefix="", nan_policy="raise", **kwargs  # noqa
    ):
        kwargs.update({"nan_policy": nan_policy, "independent_vars": independent_vars})
        if prefix != "":
            print(
                "{}: I don't know how to get prefixes working on composite models yet. "
                "Prefix is ignored.".format(self.__class__.__name__)
            )
        g1 = GaussianModelH(prefix="g1_", **kwargs)
        c = ConstantModelH(prefix="", **kwargs)
        # the below lines gives g1 + c
        super().__init__(g1, c, operator.add)
        self._set_paramhints_prefix()
        self.com_func = gaussianCH        
     def eval_fast(self, nvars, **kwargs):
        return self.com_func(kwargs['x'], *nvars)
```

where `gaussianCH` combines `GaussianModelH` and `ConstantModelH` model functions, and `eval_fast` evaluate it. In the current implementation, this is done manually. Ideally, we would like to automate this process, i.e. given any two model functions, we can generate the composite model function.

4. update `aic`, `bic` calculation (check `MinimizerResult._calculate_statistics`)
 - As suggested by the science team, the following method is used in this version:
 ```
  aic = chisqr + 2 * nvarys
  bic = chisqr + np.log(ndata) * nvarys
```
5. remove `init_fit` and `best_fit` attributes from ModelResult (check `ModelResult.fit`)
	- In DFisher_2022A, these two attributes won't be written in the final results of cube fitting. To speed up the process, here I comment out these two attributes.
