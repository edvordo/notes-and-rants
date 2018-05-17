# MutationObserver restart

_Valid as of 2018-05-17_

The default [MutationObserver](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver) object doesn't have a 
way to restart itself. You always need to call `observer.restart(target, options)` over and over again. This annoyed
me beyond believe, so ..

Here is a <sub>(hacky)</sub> way to do it yourself

## Modify the default MutationObserverObject
```javascript
MutationObserver.prototype.observeArguments = []; // internal variable to store the args 
MutationObserver.prototype.originalObserve = MutationObserver.prototype.observe; // save the original implementation
MutationObserver.prototype.observe = function(target, options) { // overwrite the function
    this.observeArguments = [target, options];
    return this.originalObserve.apply(this, this.observeArguments);
}
MutationObserver.prototype.restart = function() { // and finally add the restart function
    return this.originalObserve.apply(this, this.observeArguments);
}
```

From now on you can use your Mutation observer as you normally would

```javascript
let observer = new MutationObserver(function(mutationList) {
    mutationList.forEach(MutationRecord => {
        // do whatever you need with the MutationRecord here 
    });
    // or ..
    for (let mutation of mutationList) {
        // do whatever you need with the MutationRecord here
    }
});
observer.observe(document.querySelector('#foo-element'), {childList: true});
```


at some point you disconnect it for whatever reason

```javascript
observer.disconnect()
```

and if you ever need to **restart it again**, just call
```javascript
observer.restart()
```