# T Element

From site：http://consultrsr.net/resources/eis/diff-t.htm



The Warburg circuit element is a good place to start thinking about diffusion, but often **semi-infinite diffusion** is not a reasonable model. The **[O](http://consultrsr.net/resources/eis/diff-o.htm)** and **T** circuit elements are useful models for diffusion when a thin film ( and therefore, **finite diffusion** ) is involved.

## Diffusion Circuit Elements - T

The **T** circuit element is characteristic of another type of film -- a film which contains a fixed amount of electroactive substance. The classical "thin layer electrochemistry" cell is an example of such a system. Batteries or supercapacitors also may share this behaviour. The common feature is the fixed amount of electroactive material present. Once it has been consumed, it can not be replenished.

In the [Gamry Echem Analyst™](http://www.gamry.com/) software it is called the "bounded Warburg." This name is more descriptive of what the element really stands for.

![Nyquist Plot for a T Element](t-nyq.gif)

<center style="color:#C0C0C0">A Nyquist plot for the T element</center>

The figure  shows the Nyquist plot for the **T** diffusion element. Like the [O](http://consultrsr.net/resources/eis/diff-o.htm) element, the **T** element is characterized by two parameters,

- an "admittance" parameter, **Yo**, 
- and a "time constant" parameter, **B** (units: sec**½** ). 

At high frequency ( f > 2 / **B**<sup>2</sup> ) the **T** circuit element is indistinguishable from a Warburg impedance! This frequency range is shown in **red** in the figure. **Since the time for a molecule to diffuse across the thin layer is much longer than the period of the AC stimulus applied the electrode does not 'see' that the film or coating is of finite thickness.**

At low frequency, the **T** element looks like an R and a C in series, with R=(**B** / **Yo**) / 3.

## Equations for the T element

The equations for the complex admittance ( **Y(![omega](http://consultrsr.net/resources/eis/math/omega.gif))** ) and complex impedance ( **Z(![omega](http://consultrsr.net/resources/eis/math/omega.gif))** ) are given by the equations below. The **T** circuit element gets its name from the hyperbolic **tangent** ( tanh[] ) admittance response.

![Equations for Y and Z for a T element](http://consultrsr.net/resources/eis/math/t-eqn.gif)

**Yo** has the same [definition](http://consultrsr.net/resources/eis/warburg2.htm) as for the Warburg impedance. **Yo** can be used to calculate a diffusion coefficient for the mobile species **within** the film, coating, or thin layer cell using the same [equations](http://consultrsr.net/resources/eis/warburg2.htm). For large values of the argument (the **red** region of the Nyquist plot, above), the *tanh* and *coth* functions both approach unity and the impedance has the same ![omega](http://consultrsr.net/resources/eis/math/omega10p.gif) dependence as the Warburg. This region can be used to estimate **Yo**.

If the thickness of the thin layer is ![delta](http://consultrsr.net/resources/eis/math/delta12px.gif), then the constant B is related to this thickness and the diffusion coefficient, D. The parameter B characterizes the time it take for a reactant to diffuse from one side of the layer to the other.

![B=delta / sqrt(D)](http://consultrsr.net/resources/eis/math/b-eqn.gif)

