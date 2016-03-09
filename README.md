# limit-with-collation

```javascript
var MAX_RUNS_PER_WINDOW = 1;
var RUN_WINDOW = 1000;

/**
 * Responsibility: limits the supplied function so that
 * it can only be invoked a maximum number of times in 
 * a given time-window.
 * Calls that arrive too fast are collated ready for 
 * the next invocation of the function.
 * i.e. anything faster than the defined limit is collated.
 
 **/
function limitWithCollation(fn, options) {
    var callQueue = [], 
    invokeTimes = createCircularQ(MAX_RUNS_PER_WINDOW), 
    waitId = null;
 
    /**
     * Calls to `limited` are buffered in `callQueue`.
     * If we are within the rate limit, then we collate 
     * the calls made since the last call, and then call the 
     * function with the collation result.
     **/   
    function limited() {
        callQueue.push(arguments);
        console.log(callQueue.reduce((p,c,i) => p + c[0], ''));
        
        if (withinRateLimit()) {
            return dequeue();
        }
        
        if (waitId === null) {
            waitId = setTimeout(dequeue, timeToWait());
        }
    }
    
    limited.cancel = function() {
        clearTimeout(waitId);
    };
    
    return limited;
    
    /**     
     * Collates all calls made since the last call, and 
     * then calls the wrapped function with the 
     * collation result.
     * We only care about the last MAX_RUNS_PER_WINDOW
     * invocations, so we use a circular queue 
     * `invokeTimes` to store the last n invocations 
     * for future reference.
     **/
    function dequeue() {        
        clearTimeout(waitId); // don't think this is needed?
        waitId = null;
        invokeTimes.unshift(performance.now());
        fn.apply(null, options.collateFn(callQueue));
        callQueue.length = 0; // Empties the queue.
    }

    function withinRateLimit() {        
        return (timeForMaxRuns() >= RUN_WINDOW);
    }

    function pendingCalls() {
        return !!callQueue.length;
    }
    
    function timeToWait() {
        var ttw = RUN_WINDOW - timeForMaxRuns();
        return ttw < 0 ? 0 : ttw;
    }
    
    function timeForMaxRuns() {
        return ( performance.now() - (invokeTimes[MAX_RUNS_PER_WINDOW - 1] || 0)) ;
    }
}

function createCircularQ(length) {
    var circularQueue = [];    
    circularQueue.unshift = function(element) {
        if (this.length === length) {
            this.pop();
        }
        return Array.prototype.unshift.call(this, element);
    }
    return circularQueue;
}

// Usage example.
var printLetter = limitWithCollation(function(letter) {
    document.write(letter);
}, { collateFn: arr => [arr.reduce((p,c,i) => p + c[0] + (i === arr.length-1 ? ') ' : ', ') , '(')] }
);

var alphabet = ['A', 'B', 'C', 'D', 'E', 'F', 'G', 
'H', 'I', 'J', 'K', 'L', 'M', 'N', 
'O', 'P', 'Q', 'R', 'S', 'T', 'U', 
'V', 'X', 'Y', 'Z'], i = 0;;

!function go() {
    var curr = alphabet[i++];
    if(curr) {
        printLetter(curr);
        setTimeout(go, 100);
    }
}()

```
