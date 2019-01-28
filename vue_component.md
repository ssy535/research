# Vuejs 컴포넌트 및 필터 사용 예시

## 1. 전역 Component 사용

선언)
main.js에서 선언하고 사용
```javascript
import Currency from './views/productmanager/util/currency_'
Vue.component('Currency', Currency);
```

사용)
전역 Component이기 때문에 vue 페이지 전체적으로 
```html
<Currency v-model="unit.prdStdUprc"/>
```

## 2. 지역 Component 사용
선언)
특정 vue 페이지에서 Component 정의
```javascript
import Currency_ from "../../productmanager/util/currency_"

  export default {
    components: {
      RemoteSelect_,
      CalendarRange_,
      Currency_
    },
 }
```

사용)

```html
<Currency v-model="unit.prdStdUprc"/>
```
