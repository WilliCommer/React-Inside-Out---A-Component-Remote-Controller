# React Inside Out - A Component Remote Controller



## Contents

[TOC]



## Source Description



A Component Remote Controller consists of three parts:

1. [Event Emitter](#Event Emitter)
2. [Controller](#Controller)
3. [Component Wrapper](#Component Wrapper)





### Event Emitter

The EventEmitter is a simple pattern that allows you to create an object that emits events, and allow you to listen to those events. The `event` is a string and the `listener` is a callback function.

#### Event Listener Example

```javascript
emitter.on('componentDidMount', function() {
    console.log('componentDidMount');
});
```



We will use a simplified version of the *node* [EventEmitter](https://nodejs.org/api/events.html)




#### EventEmitter Source

```javascript
export default class EventEmitter {
    
    constructor() {
        this.events = {};
    }
    
    on (event, listener) {
        if (typeof this.events[event] !== 'object') {
            this.events[event] = [];
        }
        this.events[event].push(listener);
        return () => this.off(event, listener);
    }
    
    off (event, listener) {
        if (typeof this.events[event] === 'object') {
            const idx = this.events[event].indexOf(listener);
            if (idx > -1) {
                this.events[event].splice(idx, 1);
            }
        }
    }
    
    emit (event, ...args) {
        if (typeof this.events[event] === 'object') {
            this.events[event].forEach(listener => listener.apply(this, args));
        }
    }
    
    once (event, listener) {
        const remove = this.on(event, 
            (...args) => {
                remove();
                listener.apply(this, args);
        });
    }

};
```



### Controller



#### Controller Source

```
export class RemoteController extends EventEmitter {

    constructor () {
        super();
        this.component      = null;
        this.isMounted      = false;
        this.on('constructor',          this.onConstructor.bind(this));
        this.on('componentDidMount',    this.onDidMount.bind(this));
        this.on('componentWillUnmount', this.onUnmount.bind(this));
    }

    onConstructor (component, props) {
        this.component  = component;
        this.isMounted  = false;
    }

    onDidMount (component) {
        this.isMounted = true;
        this.emit('mount', this);
    }

    onUnmount (component) {
        this.emit('unmount', this);
        this.component  = null;
        this.isMounted  = false;
    }

    setState (props) {
        if (this.component) {
            this.component.setState(props);
            this.emit('setState', this.component);
        } 
    }

    getState () {
        return this.component ? this.component.state : false;
    }

}
```







### Component Wrapper



#### Component Wrapper Source



```
export function withController (WrappedComponent, controller) {

    return class extends React.Component {

        constructor(props) {
            super(props);
            this.state      = {...props};   // hold a copy of props in state
            this.controller = controller;   // set reference to remote controller
            this.emit( 'constructor', this, props );    // say hello
        }
            
        emit (event, ...args) {
            // emit events to remote controller
            if (this.controller)
                this.controller.emit(event, ...args);
        }
        
        componentDidMount () {
            this.emit( 'componentDidMount', this );
        }
        
        componentWillUnmount () {
            this.emit( 'componentWillUnmount', this );
            if (super.componentWillUnmount) {
                super.componentWillUnmount();
            } 
        }
        
        componentDidUpdate () {
            this.emit( 'componentDidUpdate', this );
        }
        
        render() {
            this.emit( 'render', this );	// used for debug
            return <WrappedComponent {...this.state} />;
        }
    };
}
```

