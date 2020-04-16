# 通过 NgRx 体验前端状态管理


前端中的状态指什么？前端中的状态指的是影响 UI 变化的数据。例如用户登入退出，用户的某种操作带来的 UI 视觉变化，UI 主题的切换，甚至路由的切换也是状态变化。

为什么要进行状态管理？通常来说前端中的状态是状态碎片，存在余各个组件中。这种碎片化的状态给开发团队带来的维护上的麻烦。为了规范开发流程，我们需要对状态进行集中统一管理。什么是ngrx？ngrx = rxjs + rudux。rudux是前端开发中主流状态管理库，不依赖平台。ngrx 相当于 rudux 在 angular 中的使用扩充。

> 状态管理不是前端应用必须使用的，它让简单程序变复杂，让复杂程序变简单。

angular 有无 ngrx 的架构对比：

<div style="display: flex; justify-content: space-evenly; align-items: flex-end;"><img src="https://raw.githubusercontent.com/111hunter/111hunter.github.io/master/post/resources/_gen/images/angular-mvvm.png" style="width:19.5%;" alt="angular-mvvm"><img src="https://raw.githubusercontent.com/111hunter/111hunter.github.io/master/post/resources/_gen/images/ngrx.png" style="width:65%;" alt="ngrx"></div>

<!-- <div align=center><img src="https://raw.githubusercontent.com/111hunter/111hunter.github.io/master/post/resources/_gen/images/ngrx.png" width="80%" alt="ngrx"></div> -->

没有状态管理前，构建 angular 程序我们仅需要 component, service, 以及后端数据 api 接口。而添加了状态管理后，我们的程序多出不少新的概念，状态管理将我们的应用复杂化了吗？对于复杂应用来说，是简化了。在软件开发中，一切看似的复杂往往是为了简单。其实状态管理这个概念类似于后端的数据库。

下面，我们对 ngrx 的各种新概念进行介绍，以下内容仅是个人理解。redux 中核心概念只有三个: Store, Action, Reducer，其他是 ngrx 的扩展。

## Store
简单的理解就是前端状态的数据库，它有一个基本原则: 是"唯一的、状态不可修改"的树。

## Action
可以理解为状态变化的信号，当状态发生变化时，通过 Action 显性定义状态变化。 可配合 store-devtools 使用。

## Reducer
它是前端数据状态映射成的 Store 数据库中的表，接收旧状态，返回新状态。它是一个纯函数, 需要通过 Action 告知能够进行的状态变化。

## Effects
我们在进行一些 Action 时需要发送 http 请求，DOM 操作, 读写文件等。redux 状态管理只关心视图层的状态变化, 不会解决这些的需求。因此 ngrx 扩展了 Effects, 其实它是一个钩子函数。这大概就是示意图中虚线的含义。

## Selector
性能优化，官方解释：因为选择器是纯函数，所以当参数匹配时可以返回最后的结果，而无需重新调用选择器函数。Store 通过 Selector 得到 component 状态。

## Entity
性能优化，Reducer 实现 EntityState 接口，就能够注册数据集合 id 和 entity, 通过 id 来找到数据。配合 store-devtools 工具使用。

## Router-store
侦听路由状态的变化。配合 store-devtools 工具使用。

## 整体流程

当我们 component 中的 state 需要改变时，发出 Action 信号, Reducer 根据 Action 的信号选择更新 state, Store 收到 state 的变化，通过 selector 得到状态切片选择性地更新 compont 中的 state。 有些过程无法在这个流程中实现，例如从后端 api 获取数据，于是我们在发出 Action 时，套上 Effects 钩子函数，在钩子函数中调用 service 取得后端数据。

下面是使用各个 api 的例子, 关于如何注册各个模块，官网有详细说明，不在赘述。
<br>现在我们需要管理 customer 的状态, customer 定义如下：

```ts
//customer.model.ts

export interface Customer {
  id?: number;
  name: string;
  phone: string;
  address: string;
  membership: string;
}
```

用 json.server 做服务器，创建 db.json 写入两个 customer：

```json
{
  "customers": [
    {
      "name": "John Doe",
      "phone": "910928392098",
      "address": "123 Sun Street",
      "membership": "Platinum",
      "id": 1
    },
    {
      "name": "Mary Johnson",
      "phone": "808937482734",
      "address": "893 Main Voulevard",
      "membership": "Pro",
      "id": 2
    }
  ]
}

```

我们要做的是管理监视页面加载 customer 的过程。coustom 有三种加载状态：加载中，加载成功，加载失败。
<br>先定义 customer 的状态, loading, loaded, error, 实现 EntityState 接口性能优化，方便管理：

```ts
//customer.entity.ts

import { EntityState, EntityAdapter, createEntityAdapter } from "@ngrx/entity";
import * as fromRoot from "../../state/app-state";
import { Customer } from "../customer.model";

export interface CustomerState extends EntityState<Customer> {
    selectedCustomerId: number | null;
    loading: boolean;
    loaded: boolean;
    error: string;
}

export interface AppState extends fromRoot.AppState {
    customers: CustomerState;
}

export const customerAdapter: EntityAdapter<Customer> = createEntityAdapter<
    Customer
>();

```

接着实现 Action, Action 要实现两个属性 payload? 和 type，发出 Action 的方法是 store.dispatch()。

```ts
//customer.action.ts

import { Action } from "@ngrx/store";
import { Customer } from "../customer.model";

export enum CustomerActionTypes {
  LOAD_CUSTOMERS = "[Customer] Load Customers",
  LOAD_CUSTOMERS_SUCCESS = "[Customer] Load Customers Success",
  LOAD_CUSTOMERS_FAIL = "[Customer] Load Customers Fail"
}

export class LoadCustomers implements Action {
  readonly type = CustomerActionTypes.LOAD_CUSTOMERS;
}

export class LoadCustomersSuccess implements Action {
  readonly type = CustomerActionTypes.LOAD_CUSTOMERS_SUCCESS;

  constructor(public payload: Customer[]) {}
}

export class LoadCustomersFail implements Action {
  readonly type = CustomerActionTypes.LOAD_CUSTOMERS_FAIL;

  constructor(public payload: string) {}
}

export type Action = LoadCustomers | LoadCustomersSuccess | LoadCustomersFail;
```

要从 json-server 中获取数据，我们必须实现 Effects, 调用 service:

```ts
//customer.effects.ts

import { Injectable } from "@angular/core";

import { Actions, Effect, ofType } from "@ngrx/effects";
import { Action } from "@ngrx/store";

import { Observable, of } from "rxjs";
import { map, mergeMap, catchError } from "rxjs/operators";

import { CustomerService } from "../customer.service";
import * as customerActions from "./customer.action";
import { Customer } from "../customer.model";

@Injectable()
export class CustomerEffect {
  constructor(
    private actions$: Actions,
    private customerService: CustomerService
  ) { }

  @Effect()
  loadCustomers$: Observable<Action> = this.actions$.pipe(
    ofType<customerActions.LoadCustomers>(
      customerActions.CustomerActionTypes.LOAD_CUSTOMERS
    ),
    mergeMap((actions: customerActions.LoadCustomers) =>
      this.customerService.getCustomers().pipe(
        map(
          (customers: Customer[]) =>
            new customerActions.LoadCustomersSuccess(customers)
        ),
        catchError(err => of(new customerActions.LoadCustomersFail(err)))
      )
    )
  );
}

```
接着实现 Reducer，其中的 state 显示在 store-devtools 工具中：

```ts
//customer.reducer.ts

import * as customerActions from "./customer.action";
import { CustomerState, customerAdapter } from ".";

export const defaultCustomer: CustomerState = {
  ids: [],
  entities: {},
  selectedCustomerId: null,
  loading: false,
  loaded: false,
  error: ""
};

export const initialState = customerAdapter.getInitialState(defaultCustomer);

export function customerReducer(
  state = initialState,
  action: customerActions.Action
): CustomerState {
  switch (action.type) {
    case customerActions.CustomerActionTypes.LOAD_CUSTOMERS: {
      return {
        ...state,
        loading: true
      };
    }
    case customerActions.CustomerActionTypes.LOAD_CUSTOMERS_SUCCESS: {
      return customerAdapter.addAll(action.payload, {
        ...state,
        loading: false,
        loaded: true
      });
    }
    case customerActions.CustomerActionTypes.LOAD_CUSTOMERS_FAIL: {
      return {
        ...state,
        entities: {},
        loading: false,
        loaded: false,
        error: action.payload
      };
    }

    default: {
      return state;
    }
  }
}
```

按照流程，新的数据需要在 selector 中获取：

```ts
//customer.selector.ts

import { createFeatureSelector, createSelector } from "@ngrx/store";
import { CustomerState, customerAdapter } from "./customer.entity";

const selectFeatureState = createFeatureSelector<CustomerState>(
    "customers"
);

export const selectCustomers = createSelector(
    selectFeatureState,
    customerAdapter.getSelectors().selectAll
);

export const selectCustomersLoading = createSelector(
    selectFeatureState,
    (state: CustomerState) => state.loading
);

export const selectCustomersLoaded = createSelector(
    selectFeatureState,
    (state: CustomerState) => state.loaded
);

export const selectError = createSelector(
    selectFeatureState,
    (state: CustomerState) => state.error
);
```

接着在我们的 component 中完成整个流程:

```ts
//customer-list.component.ts

import { Component, OnInit } from "@angular/core";

import { Store, select } from "@ngrx/store";
import { Observable } from "rxjs";

import { Customer } from "../customer.model";
import { AppState } from "../state/customer.entity";
import * as CustomerSelector from '../state/customer.selector'
import * as CustomerAction from '../state/customer.action'

@Component({
  selector: "app-customer-list",
  templateUrl: "./customer-list.component.html",
  styleUrls: ["./customer-list.component.css"]
})
export class CustomerListComponent implements OnInit {
  customers$: Observable<Customer[]>;

  constructor(private store: Store<AppState>) { }

  ngOnInit() {
    this.store.dispatch(new CustomerAction.LoadCustomers());
    this.customers$ = this.store.pipe(select(CustomerSelector.selectCustomers));
  }
}

```

注意此时获取的数据 customers 是 observable 对象，在 html 中需要使用 (customers$ |async) 的格式得到在 json-server 中写入的对象。

```html
<!-- customer-list.component.html -->

    <tr *ngFor="let customer of (customers$ |async)">
      <th scope="row">{{customer.name}}</th>
      <td>{{customer.phone}}</td>
      <td>{{customer.address}}</td>
      <td>{{customer.membership}}</td>
      <th>
        <a>edit</a>
        <!-- <a (click)=editCustomer(customer)>edit</a> -->
        <br>
        <a>delete</a>
        <!-- <a (click)=deleteCustomer(customer)>delete</a> -->
      </th>
    </tr>
```

当然，在 rudex-devtools 中观察调试我们能够了解到更多的技术细节。

附：[源码地址](https://gitee.com/hildakik/ngrx-demo)

**参考资料**

 - [NgRx 英文官网](https://ngrx.io)
 - [github 上一个例子](https://github.com/angulardeveloper-io/ngrx-store-app)
 
