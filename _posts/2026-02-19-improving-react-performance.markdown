---
layout: post
title: "Improving a React Application Performance by 50%"
tags: [react, performance, redux, javascript]
image: '/images/posts/react.webp'
---

In the beginning our builder at Convertflow was more than suficient for our customers. But as time passed by we've started to have more
enthusiastic and heavy customers building awesome campaigns, but bigger and complex at the same time. Some of them reaching 100 steps with
hundreds of elements, conditional logic, scripts, etc

So our core component of our funnel experimentation platform—was getting sluggish. With hundreds of child components and complex business logic running simultaneously, performance had become a real concern. After investigation and iterative improvements, we managed to achieve a 50% performance improvement.

## The Pareto Approach

When tackling performance in a large codebase, my first instinct was to apply the Pareto principle: find the 20% of changes that would deliver 80% of the results. Instead of attempting a full rewrite or touching every file, I focused on identifying the bottlenecks that had the highest impact.

Using React DevTools Profiler, Chrome Performance tools and React Scan, I traced down the components that were re-rendering the most and the operations consuming the most resources. The findings pointed to three main areas: heavily used components across the builder, canvas calculations, and state management inefficiencies.

## Improving Heavily Used Components

### Avoiding useEffect When Possible

One pattern I noticed across several components was the overuse of `useEffect`. While hooks are powerful, they can introduce unnecessary re-renders if not used carefully. In many cases, derived state or computations could be handled synchronously without side effects.

### Eliminating Anonymous Functions

Anonymous functions inside JSX cause re-renders because they create new function references on every render cycle. This was happening across our canvas selection components.

**Before:**

```javascript
<button onClick={(e) => handleClick(e, object.id)}>
  Select
</button>
```

**After:**

```javascript
const handleSelectClick = useCallback(() => {
  handleClick(object.id);
}, [object.id]);

// ...

<button onClick={handleSelectClick}>
  Select
</button>
```

By wrapping handlers in `useCallback`, we ensure stable references that don't trigger unnecessary child re-renders.

### Canvas Calculations Optimization

Our `ZoomableCanvas` component was one of the biggest offenders. It handles pan, zoom, and transform calculations in real-time. The original implementation was updating the DOM directly on every mouse or wheel event.

The improvements included:

**1. Using requestAnimationFrame for batched updates:**

```javascript
const updateCanvas = useCallback(() => {
  if (animationFrameId.current) {
    cancelAnimationFrame(animationFrameId.current);
  }

  animationFrameId.current = requestAnimationFrame(() => {
    if (zoomableDom.current) {
      zoomableDom.current.style.transform = `matrix(${currentScale.current}, 0, 0, ${currentScale.current}, ${transformX.current}, ${transformY.current})`;

      const now = Date.now();
      if (now - lastControlsUpdateTimestamp.current >= 100) {
        updateZoomControls();
        lastControlsUpdateTimestamp.current = now;
      }
    }
  });
}, [updateZoomControls]);
```

**2. Caching DOM queries:**

```javascript
const cachedZoomControls = useRef(null);

const updateZoomControls = useCallback(() => {
  if (!cachedZoomControls.current) {
    cachedZoomControls.current = document.querySelectorAll(".cf-zoom-control-percentage");
  }
  const percentage = `${Math.round(currentScale.current * 100)}%`;
  cachedZoomControls.current.forEach((el) => {
    if (el) el.textContent = percentage;
  });
}, []);
```

**3. Throttling control updates (100ms intervals):**

Instead of updating UI controls on every frame, we throttle updates to avoid unnecessary DOM manipulations.

**4. CSS performance hints:**

```css
.zoomable-canvas {
  will-change: transform;
  backface-visibility: hidden;
}
```

These properties tell the browser to optimize the element for transform operations.

## Moving Heavy Computations to Background

One of the most impactful changes was rethinking how we handle heavy operations like duplicating steps or sections. Previously, these operations were done in real-time, incrementing the DOM at each step. This caused noticeable freezes, especially for complex funnels.

The solution was to move these operations to a background job using Sidekiq and respond with WebSockets using Action Cable.

**The flow:**

1. User triggers the duplication
2. Frontend shows a loading state
3. Backend processes the duplication asynchronously in Sidekiq
4. Action Cable broadcasts the result back to the frontend
5. Frontend updates in a single batch

I've created a reusable hook for Action Cable connections:

```javascript
const useActionCable = (channel, params = {}, callbacks = {}) => {
  const {
    onConnected,
    onDisconnected,
    onReceived,
    autoDisconnect = false,
    autoDisconnectCondition = () => false,
  } = callbacks;

  const [isConnected, setIsConnected] = useState(false);
  const [lastMessage, setLastMessage] = useState(null);

  const disconnect = useCallback(() => {
    if (subscriptionRef.current) {
      subscriptionRef.current.unsubscribe();
    }
    if (cableRef.current) {
      cableRef.current.disconnect();
    }
    setIsConnected(false);
  }, []);

  useEffect(() => {
    if (!channel) return;

    cableRef.current = createConsumer();
    subscriptionRef.current = cableRef.current.subscriptions.create(
      { channel, ...params },
      {
        received(data) {
          setLastMessage(data);
          onReceived?.(data);

          if (autoDisconnect && autoDisconnectCondition(data)) {
            disconnect();
          }
        },
      }
    );

    return () => disconnect();
  }, [channel, JSON.stringify(params)]);

  return { isConnected, lastMessage, disconnect };
};
```

This approach eliminated the UI freezes entirely. The user sees a progress indicator while the work happens in the background, and the final result is applied as a single DOM update.

## Implementing Redux for Better State Management

This was probably the most significant architectural change. Our legacy code used a single React Context shared down the entire component tree. The pitfall? Even with memoized components, any change to the context caused re-renders across the tree, consuming computational resources unnecessarily.

But we didn't want to do a full rewrite and risk breaking a lot of moving pieces. The solution was to implement Redux on top of the existing state management, but only for the states that changed the most.

### The Hybrid Approach

Identifying UI states that were causing the most re-renders:

- Device selection (desktop/mobile)
- Panel state (open/closed)
- Dark mode
- Canvas lock state
- View mode
- Selected object

These states were extracted into Redux slices:

```javascript
import { createSlice } from '@reduxjs/toolkit';

const initialState = {
  device: 'desktop',
  panel_closed: false,
  dark_mode: false,
  view: 'funnel',
  zoomable_canvas_locked: true,
  new_object: {},
};

const uiSlice = createSlice({
  name: 'ui',
  initialState,
  reducers: {
    setDevice: (state, action) => {
      state.device = action.payload;
    },
    togglePanel: (state, action) => {
      state.panel_closed = action.payload;
    },
    setView: (state, action) => {
      state.view = action.payload;
    },
    setDarkMode: (state, action) => {
      state.dark_mode = action.payload;
    },
    setZoomableCanvasLocked: (state, action) => {
      state.zoomable_canvas_locked = action.payload;
    },
  },
});
```

### Store Configuration

```javascript
import { configureStore } from '@reduxjs/toolkit';
import selectionReducer from './slices/selectionSlice';
import uiReducer from './slices/uiSlice';
import objectsReducer from './slices/objectsSlice';
import historyReducer from './slices/historySlice';

export const store = configureStore({
  reducer: {
    selection: selectionReducer,
    ui: uiReducer,
    objects: objectsReducer,
    history: historyReducer,
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: false,
      immutableCheck: process.env.NODE_ENV !== 'production',
    }),
  devTools: process.env.NODE_ENV !== 'production',
});
```

### The Integration

Redux wraps the existing Context, allowing gradual migration:

```javascript
const VariantBuilder = (props) => {
  return (
    <Provider store={store}>
      <VariantBuilderContext {...props}>
        <VariantBuilderInner {...props} />
      </VariantBuilderContext>
    </Provider>
  );
};
```

Components can read from Redux for UI state while still using Context for business logic. This hybrid approach allowed us to improve performance without breaking existing functionality.

## Results and Takeaways

After implementing these changes:

- **50% reduction** in unnecessary re-renders
- **Eliminated UI freezes** during heavy operations
- **Improved perceived performance** with background processing and progress indicators

The changes were incremental and low-risk. We didn't rewrite the application; we identified the bottlenecks and addressed them surgically.

There's still plenty of room for improvement. In the daily priorities of delivering new features and fixing bugs, these optimizations were a step we made to improve our customers' experience. Sometimes the best performance wins come not from revolutionary changes, but from careful, targeted improvements.

If you're facing similar challenges with a React application, my suggestion is: profile first, identify the biggest offenders, and tackle them one at a time. The Pareto principle works.
