---
title: mobx
tags: React,mobx
grammar_cjkRuby: true
---

## stateless 层级深的component渲染条件?

```javascript
mobx.useStrict(true)

class UserStore{
	 @observable isLoadings: {} = {};
 	 @action changeLoadingState(type: string) {
    	this.isLoadings[type] = !this.isLoadings[type];
  	 }
	 @action fetchSomething(){
	 	fetch(....)
			.then(d=>{
			this.changeLoadingState('Type');
			})
	 }
}


const LoadingView = observer(({isloading})=>{
	return (isloading?"....":"more")
})

@observer
@inject('UserStore')
class News extends React.Component{
	render(){
		const { isLoadings } = this.props.UserStore
		const loading = isLoadings['Type']
		return (
			<LoadingView isloading={loading}/>
		)
	
	}
}
	
```

fuck!!!!!!!!!!!!!!!!!!!
看文档!!!!!!!!!!!!!!



**[extend-observable][1]**

## this

``` javascript 
...component/
  handleMoreClick = () => {
   const { fetchFollowers, nextHrefs } =userStore;
    const nextHref$ = nextHrefs[FETCH_FOLLOWERS];
    fetchFollowers(nextHref$);// **注意**
	}
	...store{
	@action fetchFollowers(){
		// **在这里获取this的话this指向谁?**
	 }
	}
```


[作者解释computed和observe,autorun之间的计算数据情况][2]


  [1]: https://mobx.js.org/refguide/extend-observable.html
  [2]: https://github.com/mobxjs/mobx/issues/101#issuecomment-220891704