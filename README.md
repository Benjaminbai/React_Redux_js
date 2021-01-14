# React_Redux_js
realize react_redux by js

## 实现一个 react-redux
1. 如果想要将 Redux 结合 React 使用的话，通常可以使用 react-redux 这个库
    - react-redux 一共提供了两个 API，分别是 connect 和 Provider，前者是一个 React 高阶组件
    - 后者是一个普通的 React 组件。react-redux 实现了一个简单的发布-订阅库，来监听当前 store 的变化
    - Provider：将 store 通过 Context 传给后代组件，注册对 store 的监听
    - connect：一旦 store 变化就会执行 mapStateToProps 和 mapDispatchToProps 获取最新的 props 后，将其传给子组件
2. 实现 Provider
    - 先来实现简单的 Provider，已知 Provider 会使用 Context 来传递 store，所以 Provider 直接通过 Context.Provider 将 store 给子组件
    ```
    // Context.js
    const ReactReduxContext = createContext(null);

    // Provider.js
    const Provider = ({ store, children }) => {
        return (
            <ReactReduxContext.Provider value={store}>
                {children}
            </ReactReduxContext.Provider>
        )
    }
    ```
    - Provider 里面还需要一个发布-订阅器
    ```
    class Subscription {
        constructor(store) {
            this.store = store;
            this.listeners = [this.handleChangeWrapper];
        }
        notify = () => {
            this.listeners.forEach(listener => {
                listener()
            });
        }
        addListener(listener) {
            this.listeners.push(listener);
        }
        // 监听 store
        trySubscribe() {
            this.unsubscribe = this.store.subscribe(this.notify);
        }
        // onStateChange 需要在组件中设置
        handleChangeWrapper = () => {
            if (this.onStateChange) {
            this.onStateChange()
            }
        }
        unsubscribe() {
            this.listeners = null;
            this.unsubscribe();
        }
    }
    ```
    - 将 Provider 和 Subscription 结合到一起，在 useEffect 里面注册监听
    ```
    // Provider.js
    const Provider = ({ store, children }) => {
        const contextValue = useMemo(() => {
            const subscription = new Subscription(store);
            return {
                store,
                subscription
            }
        }, [store]);
        // 监听 store 变化
        useEffect(() => {
            const { subscription } = contextValue;
            subscription.trySubscribe();
            return () => {
                subscription.unsubscribe();
            }
        }, [contextValue]);
        return (
            <ReactReduxContext.Provider value={contextValue}>
                {children}
            </ReactReduxContext.Provider>
        )
    }
    ```
3. 实现 connect
    - 使用 useContext 获取到传入的 store 和 subscription
    - 对 subscription 添加一个 listener，这个 listener 的作用就是一旦 store 变化就重新渲染组件；
    - store 变化之后，执行 mapStateToProps 和 mapDispatchToProps 两个函数，将其和传入的 props 进行合并，最终传给 WrappedComponent。
