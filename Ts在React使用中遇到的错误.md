---
title: Ts在React使用中遇到的错误
tags: Typescript,React,Tsx,Error
grammar_cjkRuby: true
---

##  Property 'styleName' does not exist on type 'HTMLProps<HTMLDivElement>'
https://github.com/gajus/react-css-modules/issues/147

> TrippBlunschi commented on Jan 4 • edited I get a similar error with
> Typescript. "Property 'styleName' does not exist on type 'HTMLProps'".
> 
> Does anyone know of a workaround?
> 
> This worked before Typescript 2.0 (by declaring a custom interface)
> but doesn't now that we use type packages i.e. npm install --save-dev
> @types/react.
> data-styleName would solve this problem, as those attributes are not
> type checked.

##   Cannot find module './dashboard.scss'

https://github.com/s-panferov/awesome-typescript-loader/issues/146#issuecomment-248808206


##  react_css_modules_1.default is not a function

``` javascript

@RModule(styles)
@inject("UserStore")
@observer
class DashBorard extends React.Component{
...}
```
在ts中,有的模块并没有default导出,所以

``` javascript
import * as  CSSModules from 'react-css-modules'
```

## react_css_modules 找不到 或者styleName 不匹配

**修改 import * as styles from '../scss'
为
const styles = require('...')**
