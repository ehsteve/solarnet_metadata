(appendix-ix)=
# Appendix IX. Higher-level data: parameterized components

One common type of higher-level data are results from analysing lower-level data by fitting of parameterized components (e.g., emission line profiles) to spectroscopic data by means of {math}`\chi^2` minimization, but so far there has been no standard mechanism for how to store such results in FITS files.

Below we describe a recommended scheme for storing such results, comprehensive enough to store any data resulting from fitting of additive and multiplicative parameterized components. The scheme allows for later manual inspection, verification, and (if desirable) modification of the results. We will refer to files using this scheme as “(SOLARNET) Type P”. We suggest that “P” is used as a suffix to the relevant data level number for such data. E.g., Solar Orbiter SPICE files using this scheme are referred to as SPICE Level 3P. _These files should be considered as a reference implementation of this recommendation_ and will be used as an example below. Below we use dimensions `[x,y,lambda,t]` simply as an example, since those are the dimensions used in SPICE Level 3P FITS files.

For a typical SPICE Level 2 data cube with dimensions `[x,y,lambda,t] = [400,400,32,100]`, fitting of a single Gaussian plus a zero-order polynomial is made for every `(x,y,t)` position. The final result is a data cube `[x,y,t,p] = [400,400,100,5]` where

- `(x,y,t,1)` is the fitted line peak intensity, {math}`I_0`
- `(x,y,t,2)` is the fitted line center {math}`\lambda_c`
- `(x,y,t,3)` is the fitted line width {math}`w`
- `(x,y,t,4)` is the fitted constant background {math}`a_0` (in a zeroth-order polynomial)
- `(x,y,t,5)` is the {math}`\chi^2` value from the fit

Thus, such SPICE Level 3P data are the best fitting parameters {math}`(\lambda;I_0,\lambda_c,w,a_0)` for the function:

```{math}
F(\lambda;I_0,\lambda_p,w,a_0)=Gaussian(\lambda;I_0,\lambda_c,w) + Polynomial(\lambda;a_0)
```
for each point `(x,y,t)`.

For readout windows with multiple significant emission lines, multiple Gaussians are used. When e.g., two Gaussians are used, the Level 3P data will be the best-fitting parameters {math}`(I_{0_1},\lambda_{p_1},w_1,I_{0_2},\lambda_{p_2}, w_2, a_0)` of the function:

```{math}
F(\lambda;I_{0_1},\lambda_{c_1},w_1,I_{0_2},\lambda_{c_2}, w_2, a_0)=Gaussian(\lambda;I_{0_1},\lambda_{c_1},w_1) + Gaussian(\lambda;I_{0_2},\lambda_{c_2},w_2) + Polynomial(\lambda;a_0)
```

for every point `(x,y,t)`, giving a Level 3P data cube with dimensions `[x,y,t,p] = [400,400,200,8]`, where (`x,y,t,1…3)` is  {math}`(I_{0_1},\lambda_{p_1},w_1)`, `(x,y,t,4..6)` is {math}`(I_{0_2},\lambda_{p_2},w_2)`, `(x,y,t,7)` is {math}`a_0`, and `(x,y,t,8)` is the {math}`\chi^2` value from the fit.

Generally, for _n_ Gaussians and a constant background, the size of the parameter dimension would be 3n+1+1. For n Gaussians and a linear background, the size would be 3n+2+1 because the last component would be {math}`Polynomial(\lambda;a_0,a_1) = a_0 + a_1\lambda`. Additional components may be defined, e.g., Voigt profiles and instrument-specific components (broadened Gauss profiles for SOHO/CDS).

Since the lambda coordinates for `(x,y,*,t)` are passed to the fitting function together with the corresponding data points, we refer to the lambda dimension as a “fitting dimension”, whereas dimensions `x`, `y`, and `t` are referred to as “result dimensions”. In principle, both the fitting dimensions and the result dimensions may be entirely different and in a completely different order for other types of data (e.g., a `STOKES` dimension may be included).

Since the lambda dimension does not appear in the resulting data cube, it is said to be “absorbed” by the fitting process.

In general, the scheme can be used to store data analysis results from fitting of any function on
the form:

```{math}
F(\lambda;\mathbf{p}) = \left( ((f_1(\lambda;\mathbf{p_1}) + f_2(\lambda;\mathbf{p_2}) + \cdots ) \cdot f_j(\lambda;\mathbf{p_j}) + \cdots \right) \cdot f_x(\lambda;\mathbf{p_x}) + \cdots
```

where {math}`f_n` are individual components, {math}`p_n` are their parameters, and {math}`p` is the aggregation of all parameters, by {math}`\chi^2` minimization of:

```{math}
\chi^2 = \sum_\lambda W(\lambda) \cdot (y(\lambda) - F(\lambda; \mathbf{p}))^2
```

where {math}`W(\lambda)` is the statistical weight of each pixel (typically {math}`1/\sigma^2` ) and {math}`y(\lambda)` is the original data. The bold font for {math}`\mathbf{p}` and {math}`\mathbf{p_n}` indicates vectors of parameters, distinguishing them from individual parameters in non-bold font.

To ensure that the result of the analysis can be interpreted correctly, the full definition of {math}`F(\lambda;\mathbf{p})` and its parameters must be specified in the header of the extension containing the result, using the following keywords:

**Mandatory general keywords for HDUs with SOLARNET Type P data**

`SOLARNET` must be set to either `0.5` or `1`, and `OBS_HDU``=2` _(not `1`!)_ signals that the HDU contains SOLARNET Type P data.

`ANA_NCMP` must be set to the number of components used in the analysis.

The `CTYPEi` of the parameter dimension must be `'PARAMETER'`. Note that the Meta-HDU mechanism ([Appendix III](#appendix-iii)) may be used to split Type P data over multiple files along this dimension, so e.g., parameters from each component are stored in separate files. In such cases, all HDUs should contain a full complement of all keywords defined here (including those describing components whose parameters are not present in the file).

**Mandatory keywords describing each component**

`CMP_NPn`: Number of parameters for component `n`

`CMPTYPn`: Component type, e.g.,
- `'Polynomial'`, a polynomial {math}`p_1 + p_2 \lambda + p_3 \lambda^2 + \dotsi` of order `CMP_NPn - 1`.
- `'Gaussian'`, a Gaussian {math}`p_1 e^{-1/2(\lambda - p_2)^2/p_3^2}`
- `'SSW comp_bgauss'`, a broadened Gaussian {math}`f(p_1, \dots, p_5)`, see SSW routine `comp_bgauss`
- `'SSW comp_voigt'`, a Voigt profile {math}`f(p_1, \dots, p_4)`, see SSW routine `comp_voigt`
- … etc. If you need additional component types, create an [issue](https://github.com/IHDE-Alliance/solarnet_metadata/issues).

**Optional _functional_ keywords for each component**

`CMPMULn`: Indicates whether component n is multiplicative (`CMPMULn=1`), in which case it is to be applied to the result of all previous components `n-1`, `n-2`, etc. This can be used for e.g., extinction functions. Default value is `0`.

`CMPINCn`: Indicates whether component `n` is included (`CMPINCn=1`) or excluded (`CMPINCn=0`) in the fit. This allows specification of components that would normally be included but for some reason (e.g., poor S/N ratio) has been left out for this particular data set. If `CMPINCn` is zero, additive components have a value of zero independent of the parameter values, and multiplicative components have a value of 1. Default value is 1, i.e., the component is included in the fit. The parameters in the data cube should be set to the initial values that would have been used if the component was included. If this is not feasible, the parameters should be set such that the component value would be zero if it had in fact been included.

**Optional _descriptive_ keywords for each component**

`CMPNAMn`: Name of component `n`, typically used to identify/label the emission line fitted, e.g., `'AutoGauss79.5'`, `'He I 584'`.

`CMPDESn`: Description of component `n`

`CMPSTRn`: Used by SSW’s `cfit` routines for the string to be included in composite function names. E.g., the function string for a Gaussian is `'g'`, and `'p1'` for a first-order polynomial. An automatically generated composite function of the two would be called `'cf_g_p1_'`.

Each component can have up to 26 parameters labelled with `a=A-Z`, corresponding to
component parameters {math}`p_a = p_1 \dots p_{26}`.

**Mandatory functional keyword for each parameter**

`PUNITna`: The units for parameter a of component `n`, specified according to the FITS Standard Section 4.3, e.g., `'nm'` or `'km/s'`.

**Optional functional keywords for each parameter**

`PINITna`: Initial value for {math}`p_a` used during fitting

`PMAXna`: Maximum value ({math}`p_a` has been clamped to be no larger than `PMAXna` during fitting)

`PMINna`: Minimum value ({math}`p_a` has been clamped to be no less than `PMINna` during fitting)

`PCONSna`: Set to 1 if {math}`p_a` has been kept constant during fitting

`PTRAna`: Linear transformation coefficient {math}`A` (default value 1), see below

`PTRBna`: Linear transformation constant {math}`B` (default value 0), see below

When `PCONSna``=1`, i.e., when {math}`p_a` has been kept constant during fitting, it does not mean that {math}`p_a` necessarily has the same value for all `(x,y,t)`. The parameter may have been set to different values at different points prior to the fitting e.g., manually, and then not been allowed to change during subsequent fitting of the other parameters. For points where a parameter has been kept constant, the `PINITna` value does not apply. Using the example above, the data cube value for `(x,y,t,p)` can differ from `(x,y,t+1,p)` even if the corresponding `PCONSna` value is `1`. It is also possible to keep a parameter constant only at specific points `(x,y,t)` using a constant mask in a separate extension with the same dimensionality as the result data cube (except for the last dimension, which will be one smaller than in the result data cube because there is no constant mask for the {math}`\chi^2` value). If the constant mask extension is present, parameter number `p` has been kept constant/fixated for `(x,y,t)` at the value given in the result data if and only if the constant mask `(x,y,t,p)=1`. Thus, values in the constant mask overrides the `PCONSna` value.

A Gaussian component is explicitly defined to be simply {math}`f(\lambda;p_1,p_2,p_3)=p_1 e^{-1/2(\lambda - p_2)^2/p_3^2}`. However, some may prefer to store results in modified form, such as velocities instead of line positions, and with varying definitions of line width (e.g., FWHM). To accommodate this without having to create separate components for every form, it is possible to use the `PTRAna` and `PTRBna` keywords to define a linear transformation between the _nominal_ (stored) value {math}`n_a`) of a parameter and the actual value {math}`p_a`) that is passed to the component function.

Given {math}`A_a`=`PTRAna` and {math}`B_a`=`PTRBna`, the actual parameter value passed to the component function is {math}`p_a = A_a \cdot n_a + B_a` and conversely {math}`n_a = \frac{p_a - B_a}{A_a}`. If we set {math}`A = \frac{\lambda_0}{c}` and {math}`B = \lambda_0` then:

```{math}
n_a = \frac{c(p_a - \lambda_0)}{\lambda_0}
```

Thus, if {math}`p_a = \lambda_c` (the fitted line centre) then the nominal value stored in the data cube is {math}`n_a = v` (the line velocity, with positive values for red shifted lines).

Likewise for the third parameter of a Gaussian, if {math}`A = \frac{1}{2\sqrt(2ln2)}` then the nominal value stored in the data cube will be the full width of half maximum (FWHM).

**Optional descriptive keywords for each parameter**

`PNAMEna`: Parameter name, e.g., `'intensity'`, `'velocity'`, `'width'`

`PDESCna`: Parameter description

**Optional functional keywords for the analysis as a whole**

`PGFILENA`: the name of the (progenitor) file containing the original data

`PGEXTNAM`: the name of the extension in the progenitor file that contains the original data

`NXDIM`: the number of dimensions absorbed by the fitting process

`XDIMTYm`: the `CTYPEi` of the m<sup>th</sup> dimension absorbed by the fitting process.

To allow full manual inspection, verification, and modification of the analysis results, several auxiliary data arrays may be stored in separate HDUs, with their `EXTNAME` given in the following keywords. In the description we specify the dimensionalities that would result from the example discussed above.

- `RESEXT`: the HDU containing the analysis results (`[x,y,t,p]`). Note `OBS_HDU=2`
- `DATAEXT`: the original data/Obs-HDU (`[x,y,lambda,t]`)
- `WGTEXT`: data weights  used during fitting (`[x,y,lambda,t]`). When not present, all data points are assumed to have equal weight.
- `RESIDEXT`: residuals from the fitting process (`[x,y,lambda,t]`) which may in some cases be an important factor in the verification e .g., to discover emission lines that have not been considered during the fitting
- `CONSTEXT`: constant mask (`[x,y,t,p]`) – if the constant mask value `(x,y,t,p)=1`, parameter `p` has been kept constant/frozen at the stored value during the fitting process for point `(x,y,t)`. When the constant mask extension is not present, it is assumed that all parameters have been fitted freely (between the specified min and max values) at all times unless `PCONSna=1`.
- `INCLEXT`: component inclusion mask (`[x,y,t,n]`) – if `(x,y,t,n)=0`, component `n` has not been included for point `(x,y,t)`. When not present, it is assumed that all components have been included at all times.
- `XDIMXTm`: The values of coordinates absorbed by the fitting process (`[x,y,lambda,t]`) specified by `XDIMTYm`, (see above) may be included in separate extensions as a convenience, although this is redundant whenever any of the arrays with all dimensions of the original data is present and contains the appropriate WCS information. Thus, `XDIMXTm` refers to the extension containing absorbed coordinate number `m` (with coordinate specification given by `XDIMTYm`).

In all such extensions, all WCS keywords that apply must be present, given their dimensionalities, as must all Type P-related keywords (including e.g., the extension names and component/parameter descriptions etc., and `OBS_HDU=2` as these are also “type P” data). For the component inclusion mask extension (`INCLEXT`), the `CTYPEi` of the component dimension should be `'COMPONENT'`.

For these auxiliary extensions, it may be worth considering the “external extensions” mechanism, see [Appendix VII](#appendix-vii).

At the time of writing, the SPICE Level 3P pipeline is not yet set in stone, and no Level 3P data has been delivered to the Solar Orbiter archive, thus there is no lock-in of the definitions yet. Please inform us by creating an [issue](https://github.com/IHDE-Alliance/solarnet_metadata/issues) if you implement this mechanism.

**Extension to other types of higher-level data**

The Type P storage scheme may also be used for results from other types of analyses that do not involve forward modelling of the data and subsequent {math}`\chi^2` minimisation, as a way to store interrelated parameters that have been determined from data in other ways, e.g. Mg II k line parameters, with `CMPNAMn='Mg II k'`, and `PNAMEna` set to e.g., `'k2v'`, `'k2r'`, or `'k3'`. For such cases, other values for the `CMPTYPn` keywords must be found (add an [issue](https://github.com/IHDE-Alliance/solarnet_metadata/issues)), and the size of the `PARAMETER` dimension will be equal to the sum of the `CMP_NPn` keywords, not the sum plus 1 as is normally the case.
