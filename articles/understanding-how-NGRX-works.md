### What is NGRX and how does it work?

NGRX is a Redux-based library implemented for Angular that focuses on managing application state reactively. Its architecture is based on key concepts such as actions, reducers, effects, stores, and selectors. The workflow in NGRX is cyclical and reactive: when an action is performed, the state is modified and the components that depend on it are notified, allowing them to update automatically without having to manage the state directly in each one.

To understand it better, let's see how each element of NGRX works:

* Actions: These are simple objects that describe an event. For example, “userLogin” or “loadProductList” are actions that notify changes in the application.
* Reducers: These function as pure functions that receive an action and the current state of the application and return a new state, thus updating the “Store.”
* Store: Stores the state and allows all components to access it through selectors.
* Selectors: Functions that extract specific data from the “Store” to facilitate its use in components.
* Effects: Allow asynchronous operations (such as HTTP requests) to be handled separately, intercepting actions to perform, for example, API calls and then triggering an action with the result.

### How does the NGRX workflow work?

To better visualize this, here is an illustration of how NGRX handles the workflow in Angular:

1\. The user performs an action on the interface (for example, they want to add a product to the shopping cart).

2\. This action triggers an NGRX action (e.g., addProductToCart).

3\. The action reaches the reducer, where the cart status in the store is updated.

4\. If necessary, an effect intercepts this action, makes an HTTP request to an API, and returns the result to the store.

5\. Components, which listen to the state through selectors, automatically receive the changes and update in the UI.

![NGRX flow](/assests/ngrx-flow.png)

### Example

Now let's look at a very simple [example](https://github.com/diegofr-98/cart-ngrx "NGRX cart example") of how to manage the state of what would be a shopping cart.

### 1. Create Angular project

```shell
ng new cart-ngrx
cd cart-ngrx
```

### 2. Install NGRX dependencies

```shell
ng add @ngrx/store@latest
ng add @ngrx/effects@latest
ng add @ngrx/store-devtools@latest
```

### 3. Define data model

Folder: src/app/models/product.model.ts

```typescript
export interface Product {
  id: number;
  name: string;
  price: number;
}

export interface CartItem {
  product: Product;
  quantity: number;
}
```

### 4. Define initial state

Folder: src/app/state/cart.state.ts

```typescript
import { CartItem } from '../models/product.model';

export interface CartState {
  items: CartItem[];
}

export const initialCartState: CartState = {
  items: [
    {
      product: {
        id: 1,
        name: 'Product 1',
        price: 100
      },
      quantity: 1
    },
    {
      product: {
        id: 2,
        name: 'Product 2',
        price: 200
      },
      quantity: 2
    },
  ]
};
```

### 5. Define NGRX actions

Folder: src/app/state/cart.actions.ts

```typescript
import { createAction, props } from '@ngrx/store';
import { Product } from '../models/product.model';

export const addProduct = createAction(
  '[Cart] Add Product',
  props<{ product: Product }>()
);

export const removeProduct = createAction(
  '[Cart] Remove Product',
  props<{ productId: number }>()
);

export const clearCart = createAction('[Cart] Clear Cart');

```

### 6. Create reducer

Folder: src/app/state/cart.reducer.ts

```typescript
import { createReducer, on } from '@ngrx/store';
import { addProduct, removeProduct, clearCart } from './cart.actions';
import { initialCartState } from './cart.state';

export const cartReducer = createReducer(
  initialCartState,
  on(addProduct, (state, { product }) => {
    const existingItem = state.items.find(item => item.product.id === product.id);
    const updatedItems = existingItem
      ? state.items.map(item =>
        item.product.id === product.id
          ? { ...item, quantity: item.quantity + 1 }
          : item
      )
      : [...state.items, { product, quantity: 1 }];

    return { ...state, items: updatedItems };
  }),
  on(removeProduct, (state, { productId }) => ({
    ...state,
    items: state.items.filter(item => item.product.id !== productId)
  })),
  on(clearCart, state => initialCartState),
  // on(clearCart, state => ({...state, items: []}))
);
```

### 7. Create Cart component

```shell
ng generate component cart
```

### 8. Add logic to the component

```typescript
import { Component } from '@angular/core';
import { Store } from '@ngrx/store';
import { Observable } from 'rxjs';
import { addProduct, removeProduct, clearCart } from '../state/cart.actions';
import { CartState } from '../state/cart.state';
import { Product, CartItem } from '../models/product.model';
import { AsyncPipe } from '@angular/common';

@Component({
  selector: 'app-cart',
  standalone: true,
  imports: [AsyncPipe],
  templateUrl: './cart.component.html',
  styleUrl: './cart.component.css'
})
export class CartComponent {
  cartItems$: Observable<CartItem[]>;
  constructor(private store: Store<{ cart: CartState }>) {
    this.cartItems$ = store.select(state => state.cart.items);
  }

  addProduct(product: Product) {
    this.store.dispatch(addProduct({ product }));
  }

  removeProduct(productId: number) {
    this.store.dispatch(removeProduct({ productId }));
  }

  clearCart() {
    this.store.dispatch(clearCart());
  }
}
```

### 9. And finally, the template

```html
@if (cartItems$ | async; as cartItems) {
  @for (item of cartItems; track item.product.id) {
    <p>{{ item.product.name }} - {{ item.quantity }} - ${{ item.product.price * item.quantity }}</p>
    <button (click)="addProduct(item.product)">Añadir</button>
    <button (click)="removeProduct(item.product.id)">Eliminar</button>
  }
}

<br>
<button (click)="clearCart()">Vaciar Carrito</button>
```

### When to Use NGRX and When Not to?

NGRX is useful but not always necessary. Whether to use it depends on the complexity of the application and how many components require centralized data.

When to use NGRX:

* When you are dealing with a complex application and want a centralized state.
* When multiple components need to access and modify shared data.
* When you have a complex data flow with multiple asynchronous actions and effects.

When to avoid NGRX:

* In small applications with little or no shared data flow.
* If the application is simple and the state can be managed locally in the components.
* When the time and complexity required to learn and configure a solution outweighs the improvement it brings to state management.