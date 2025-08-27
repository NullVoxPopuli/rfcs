---
stage: accepted
start-date: # In format YYYY-MM-DDT00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - data
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: # Fill this in with the URL for the Proposal RFC PR
project-link:
suite: 
---

<!--- 
Directions for above: 

stage: Leave as is
start-date: Fill in with today's date, 2032-12-01T00:00:00.000Z
release-date: Leave as is
release-versions: Leave as is
teams: Include only the [team(s)](README.md#relevant-teams) for which this RFC applies
prs:
  accepted: Fill this in with the URL for the Proposal RFC PR
project-link: Leave as is
suite: Leave as is
-->

<!-- Replace "RFC title" with the title of your RFC -->

# Universal Reactivity: Framework-Agnostic Reactive Primitives for Ember

## Summary

Modern applications increasingly combine multiple technologies - Ember for the main app, React for visualizations, Vue for specific widgets, or non-DOM renderers for CLI tools, canvas graphics, and WebGL. These different rendering targets can't share reactive state seamlessly.

This RFC proposes universal reactivity primitives that work with any renderer. Reactive cells and formulas work whether you're rendering to DOM, canvas, terminal, or command-line. The reactive state becomes the universal language connecting different rendering strategies.

This builds on Starbeam's framework-agnostic reactive primitives but goes further - enabling any renderer to participate in a shared reactive ecosystem. Ember becomes the connective tissue between different rendering technologies.

**Note on Existing APIs**: Universal reactivity complements existing Ember patterns. `@tracked` and `@cached` remain the preferred approach for most development and continue working exactly as today. The lower-level primitives (`Cell`, `Formula`, reactive collections) serve as advanced tools for decorator-free environments, framework integrations, and non-DOM rendering.

## Related RFCs

Universal reactivity complements several RFCs currently in progress:

**[Cell RFC (#1071): "A new reactive primitive: `cell`"](https://github.com/emberjs/rfcs/pull/1071)** by NullVoxPopuli (S-Exploring stage) - Proposes a new reactive primitive that provides the foundation for manual reactive state management with APIs like `cell.current`, `cell.set()`, and `cell.update()`. The Cell primitive supersedes the tracked storage primitives RFC and provides the low-level building block for reactive state that universal reactivity uses in its bridges.

**[Resources RFC (#1122): "Resources"](https://github.com/emberjs/rfcs/pull/1122)** by NullVoxPopuli (S-Proposed stage) - Adds resource management with lifecycle and cleanup capabilities built on reactive primitives. Resources provide a higher-level abstraction for managing stateful operations with proper setup and teardown that integrates seamlessly with universal reactivity patterns.

**[lowLevel.subtle.sync RFC (#1136): "lowLevel.subtle.sync"](https://github.com/emberjs/rfcs/pull/1136)** by NullVoxPopuli (S-Proposed stage) - Provides synchronization primitives for external state management, previously called "watch". This enables reactive integration with external systems like localStorage, WebSocket connections, and browser APIs - exactly the use case that universal reactivity's `Sync()` primitive addresses.

These RFCs together form a comprehensive reactivity system that enables universal reactive programming patterns across different environments and frameworks.

## Motivation

The most interesting applications being built today don't fit neatly into the boundaries of a single framework. I've worked with teams building data visualization dashboards where the main interface is built in Ember, but the charts themselves are React components because that's where the best visualization libraries live. I've seen command-line tools that need to share business logic with web applications, and desktop applications that want to reuse reactive state management from their web counterparts.

Today's reactive systems are designed around specific rendering targets - Ember's tracking system is optimized for DOM updates, React's state management is built for component re-rendering, and Vue's reactivity is tuned for template-based updates. But modern applications increasingly need to share reactive logic across different environments: a web interface, a CLI tool, a data visualization rendered to canvas, or background workers that process data. What if reactive state could flow naturally between a web interface, a CLI tool, and a canvas-based data visualization, all sharing the same underlying logic?

This enables reactive programming for any rendering target. Reactive primitives can drive DOM updates, canvas redraws, terminal interfaces, WebGL scenes, or file system operations. The reactive logic becomes the foundation that powers whatever interface your application needs.

Traditional reactive systems are tightly coupled to specific rendering targets. But reactive programming as a paradigm is more general - the core insight that changes automatically propagate to dependent computations applies whether you're updating a web page or refreshing a terminal display.

Starbeam demonstrated that reactive primitives can be decoupled from rendering concerns. A reactive cell doesn't need to know whether its changes trigger DOM updates, canvas redraws, or CLI refreshes. This separation enables truly universal reactive logic.

This opens opportunities for:
- Data management libraries that work in web apps, CLI tools, and desktop applications
- Component libraries that render to DOM, canvas, terminals, or static files  
- Debugging tools that visualize reactive state regardless of consumers

Following the warp-drive model, reactive primitives become the universal substrate for any system that understands the protocols. You can integrate with existing reactive libraries and build bridges between different reactive systems without losing benefits.

Ember becomes the universal connector - the framework that speaks all reactive dialects and ties together whatever technologies your application needs.

## Detailed design

Universal reactivity provides a different approach to reactive state that opens new possibilities while building on existing foundations. Here's how it works, from primitives to advanced behaviors.

### Cells: Storage That Knows When It Changes

A Cell holds a value and tracks when it changes. The simple API handles advanced features under the hood.

```js
import { Cell } from '@ember/reactivity';

const userName = Cell('Alice');
console.log(userName.current); // 'Alice'

userName.current = 'Bob';
console.log(userName.current); // 'Bob'
```

What makes this interesting isn't the API - it's what happens when you try to set the same value twice. The cell implements an equivalence check that determines whether the new value is actually different from the old one. If they're equivalent, the cell doesn't emit any change notifications, which means dependent computations don't rerun unnecessarily.

```js
const settings = Cell({ theme: 'dark', language: 'en' });

// This won't trigger any updates - the object is equivalent
settings.current = { theme: 'dark', language: 'en' };

// But this will, because the theme actually changed
settings.current = { theme: 'light', language: 'en' };
```

By default, cells use `Object.is` for equivalence checking, which works well for primitive values and catches the common case where you're setting a property to the same value it already has. But for objects, you often want more detailed comparison logic:

```js
const userSettings = Cell(
  { theme: 'dark', notifications: true },
  { 
    equals: (oldVal, newVal) => 
      oldVal.theme === newVal.theme && 
      oldVal.notifications === newVal.notifications 
  }
);
```

This equivalence system is crucial for performance because it prevents unnecessary updates. With cells and custom equivalence functions, you can be precise about what actually counts as a change.

Functional updates provide atomic updates based on the current value:

```js
const counter = Cell(0);
counter.update(current => current + 1);

const todoList = Cell([]);
todoList.update(todos => [...todos, { id: Date.now(), text: 'New item' }]);
```

They ensure correct equivalence checks and composability for advanced reactive patterns.

Cells can be frozen to prevent updates and eliminate tracking overhead:

```js
const apiConfig = Cell({ baseUrl: 'https://api.example.com' });
// After app initialization...
apiConfig.freeze();
```

Frozen cells have zero ongoing performance overhead since the reactive system knows they'll never change.

### Formulas: Computations That Track Dependencies Automatically

Formulas provide reactive computation without requiring dependency declarations. They track dependencies by watching what gets accessed during execution.

```js
const firstName = Cell('Alice');
const lastName = Cell('Bob');
const fullName = Formula(() => `${firstName.current} ${lastName.current}`);

console.log(fullName.current); // 'Alice Bob'
firstName.current = 'Charlie';
console.log(fullName.current); // 'Charlie Bob'
```

Automatic dependency tracking establishes a "tracking frame" during formula execution. Any accessed cell or formula becomes a dependency, solving the maintenance problem of explicit dependency lists.

This enables patterns that would be awkward with explicit declarations:

```js
const user = Cell({ name: 'Alice', preferences: { showDetails: false } });
const publicInfo = Cell({ bio: 'Software developer', location: 'San Francisco' });
const privateInfo = Cell({ email: 'alice@example.com', phone: '555-1234' });

const displayInfo = Formula(() => {
  const userData = user.current;
  const baseInfo = publicInfo.current;
  
  if (userData.preferences.showDetails) {
    // Only creates dependency on privateInfo when showDetails is true
    return { ...baseInfo, ...privateInfo.current };
  } else {
    // When showDetails is false, privateInfo changes won't trigger updates
    return baseInfo;
  }
});
```

This conditional dependency tracking is powerful because the formula's dependency set changes based on the current state of the system. When `showDetails` is false, changes to `privateInfo` won't cause `displayInfo` to recompute, even though the formula contains code that could access `privateInfo`. Only when `showDetails` becomes true does `privateInfo` join the dependency set.

The system includes two variants of formulas that handle different performance scenarios. Regular formulas recompute on every access, which makes them suitable for environments where some dependencies might exist outside the reactive system:

```js
// Regular formula - always recomputes when accessed
const mixedDependencies = Formula(() => {
  // This might change for reasons outside the reactive system
  const externalState = window.someGlobalState;
  return externalState + reactiveCell.current;
});
```

Cached formulas only recompute when their reactive dependencies change, which provides better performance for pure reactive computations:

```js
// Cached formula - only recomputes when dependencies change
const pureDependencies = CachedFormula(() => {
  return cellA.current * cellB.current + cellC.current;
});
```

The choice between regular and cached formulas becomes important when you're integrating with other frameworks or libraries that have their own reactive systems. Regular formulas ensure that external changes trigger recomputation even if they're not visible to the reactive dependency tracking.

### Choosing the Right Level of Abstraction

For most development, **`@tracked` and `@cached` should be your first choice**, whether you're building in Ember or other environments. These decorators aren't Ember-specific - they're convenient wrappers around the universal reactivity primitives and work in any JavaScript environment that supports decorators.

```js
import { tracked } from '@glimmer/tracking';
import { cached } from '@glimmer/tracking';

// Works in Ember, Node.js, vanilla JS, or any other environment
export default class ShoppingCart {
  @tracked items = [];
  @tracked discountCode = '';
  
  @cached
  get subtotal() {
    return this.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }
  
  @cached
  get discount() {
    return this.discountCode === 'SAVE10' ? 0.1 : 0;
  }
  
  @cached
  get total() {
    return this.subtotal * (1 - this.discount);
  }
}
```

The lower-level primitives like `Cell`, `Formula`, and reactive collections should be considered **advanced tools for specific scenarios**:

1. **Environments without decorator support**: Where you can't use `@tracked` and `@cached`
2. **Building framework integrations**: When bridging to other reactive systems like MobX or Vue
3. **Performance-critical code**: Where you need fine-grained control over invalidation patterns
4. **Library authoring**: When building reusable reactive abstractions that need maximum flexibility

```js
// Lower-level primitives for advanced scenarios
import { Cell, Formula, reactive } from '@ember/reactivity';

// This might be used in a library that bridges Ember with other frameworks
export class ReactiveDataAdapter {
  #source = Cell(null);
  #transformations = reactive.Array([]);
  
  output = Formula(() => {
    let data = this.#source.current;
    for (let transform of this.#transformations) {
      data = transform(data);
    }
    return data;
  });
}
```

The key insight is that `@tracked` and `@cached` are built on universal reactivity primitives, providing all the benefits (precise invalidation, performance optimizations, debugging) while maintaining familiar Ember patterns.

### Understanding the Reactive Primitives

```js
const x = Cell(2);
const y = Cell(3);
const sum = Formula(() => x.current + y.current);
const product = Formula(() => x.current * y.current);
const ratio = Formula(() => sum.current / product.current);
```

What's notable about this composition is that changes propagate efficiently through the graph. When `x` changes, `sum`, `product`, and `ratio` all get marked as potentially stale, but they only recompute when their values are actually needed. This lazy evaluation prevents unnecessary work while ensuring computed values are always current when accessed.

### The Tag System: Precise Change Tracking

The tag system enables efficient reactivity. Every reactive value has a tag that tracks revision history and dependencies.

```js
import { TAG } from '@ember/reactivity';

const cell = Cell(42);
const tag = cell[TAG];
console.log(tag.lastUpdated); // Some revision number

cell.current = 43;
console.log(tag.lastUpdated); // A higher revision number
```

Cell tags track the global revision number when last modified. The system maintains a global counter that increments on every state change, providing total ordering of mutations and enabling optimizations.

Formula tags aggregate information from dependencies:

```js
const a = Cell(1);
const b = Cell(2);
const sum = Formula(() => a.current + b.current);

const sumTag = sum[TAG];
// sumTag.dependencies() returns current dependency tags
// sumTag.lastUpdated reflects when any dependency last changed
```

The tag system enables key optimizations:

1. **Batch processing**: Mark multiple values as stale, then resolve efficiently when needed
2. **Precise invalidation**: Track which parts of large object graphs are accessed and only invalidate when those parts change  
3. **Rich debugging**: Maintain history of dependencies and explain why computations ran

The revision-based approach handles edge cases - if a formula depends on two cells that both change in the same cycle, the formula recomputes once with latest values rather than twice with intermediate states.

### Reactive Collections: Fine-Grained Invalidation

Reactive collections understand different interaction patterns and create dependencies based on actual usage, rather than treating arrays/objects as monolithic entities.

```js
import { reactive } from '@ember/reactivity';

const users = reactive.Map();
const userCount = Formula(() => users.size);
const hasAdminUser = Formula(() => users.has('admin'));
const adminUserName = Formula(() => users.get('admin')?.name);

users.set('alice', { name: 'Alice', role: 'user' });
// Only userCount is invalidated - hasAdminUser and adminUserName are unaffected

users.set('admin', { name: 'Bob', role: 'admin' });
// All three formulas are invalidated

users.set('admin', { name: 'Robert', role: 'admin' });
// Only adminUserName is invalidated - userCount and hasAdminUser are unaffected
```

This granular invalidation is possible because the reactive collections track different types of access patterns separately. When you call `has()` on a reactive map, it creates a dependency on the membership of that specific key. When you call `get()`, it creates a dependency on the value stored at that key. When you iterate over the collection, it creates dependencies on the iteration protocol.

The implications for performance are substantial. With reactive collections, only computations that actually care about what changed get invalidated.

Consider a real-world example - a todo list application:

```js
const todos = reactive.Array([
  { id: 1, text: 'Buy milk', completed: false },
  { id: 2, text: 'Walk dog', completed: true }
]);

const todoCount = Formula(() => todos.length);
const completedCount = Formula(() => todos.filter(t => t.completed).length);
const todoTexts = Formula(() => todos.map(t => t.text));

// Adding a new todo only invalidates todoCount
todos.push({ id: 3, text: 'Clean house', completed: false });

// Changing completion status only invalidates completedCount
todos[0].completed = true;

// Changing text only invalidates todoTexts
todos[1].text = 'Walk the dog';
```

This level of granularity eliminates a lot of unnecessary computation in typical applications, but it requires careful implementation to track all the different access patterns correctly. The reactive collections need to understand not just reads and writes, but also iteration patterns, length checks, and methods like `find()`, `some()`, and `every()`.

### Integration with Ember's Component Model

Universal reactivity integrates naturally with Ember components, templates, and the rendering system. The integration feels familiar while enabling new patterns:

Component integration starts with replacing `@tracked` properties and computed properties with cells and formulas:

```js
class UserProfile extends Component {
  // Replace @tracked properties with cells
  #isEditing = Cell(false);
  #draftName = Cell('');
  
  // Replace computed properties with formulas
  displayName = Formula(() => {
    if (this.#isEditing.current) {
      return this.#draftName.current || 'Enter name...';
    }
    return this.args.user.name;
  });
  
  // Actions can manipulate cells directly
  @action
  startEditing() {
    this.#isEditing.current = true;
    this.#draftName.current = this.args.user.name;
  }
  
  @action
  saveChanges() {
    if (this.#draftName.current.trim()) {
      this.args.onNameChange(this.#draftName.current);
    }
    this.#isEditing.current = false;
  }
}
```

Template integration requires updates to the rendering system to understand reactive values, but the changes should be largely invisible to developers:

```hbs
{{!-- Templates automatically unwrap reactive values --}}
<div class="user-name">{{this.displayName.current}}</div>

{{!-- Or with a helper for cleaner syntax --}}
<div class="user-name">{{reactive this.displayName}}</div>

{{#if (reactive this.isEditing)}}
  <input value={{reactive this.draftName}} {{on "input" this.updateDraft}} />
  <button {{on "click" this.saveChanges}}>Save</button>
{{else}}
  <button {{on "click" this.startEditing}}>Edit</button>
{{/if}}
```

The rendering system establishes tracking contexts during template rendering so that dependencies on reactive values are properly recorded with precise tracking of accessed reactive values.

### Resource-Based Renderer Architecture

Universal reactivity enables a fundamentally different approach to component implementation using Resources to manage DOM node lifecycle. Rather than using traditional components, the renderer implements components as Resource-managed reactive systems.

#### Cell-Optimized DOM Rendering

Cell API features provide specific optimizations for DOM rendering:

**Freeze Optimization**: `Cell.freeze()` tells the renderer to stop tracking a value entirely. This is particularly powerful for DOM elements that reach a final state:

```javascript
const DOMNodeResource = Resource(({ on }) => {
  const element = Cell(document.createElement('div'));
  const isInitialized = Cell(false);
  
  on.setup(() => {
    // Configure DOM element
    element.current.className = 'my-component';
    element.current.id = generateUniqueId();
    
    // Once DOM setup is complete, freeze these values
    element.freeze(); // Renderer stops tracking this element reference
    isInitialized.set(true);
    isInitialized.freeze(); // No more initialization checks needed
  });
  
  return { element: element.current, ready: isInitialized.current };
});
```

**Custom Equality for DOM Properties**: Cell equality functions prevent unnecessary DOM operations:

```javascript
const styleCell = Cell(
  { color: 'red', fontSize: '14px' },
  { 
    equals: (oldStyle, newStyle) => 
      oldStyle.color === newStyle.color && 
      oldStyle.fontSize === newStyle.fontSize 
  }
);

// DOM only updates when style properties actually change
const StyleResource = Resource(({ on }) => {
  const element = use(DOMNodeResource).element;
  
  return Formula(() => {
    const style = styleCell.current;
    Object.assign(element.style, style);
    return style;
  });
});
```

**Side-Signal Pattern for Event Handling**: Cells can track DOM state without storing the actual values:

```javascript
const ClickResource = Resource(({ on }) => {
  const element = use(DOMNodeResource).element;
  const clickSignal = Cell(null, { equals: () => false }); // Always dirty
  
  on.setup(() => {
    element.addEventListener('click', () => {
      clickSignal.set(null); // Signal click occurred without storing event
    });
  });
  
  return Formula(() => {
    clickSignal.current; // Consume signal to register dependency
    return handleClick(); // Execute click handler
  });
});
```

#### Component Implementation Through Resources

Traditional components are replaced by Resource compositions that manage DOM lifecycle:

```javascript
// Instead of a Component class, implement as Resource composition
const ButtonComponent = Resource(({ args, on }) => {
  const element = use(DOMNodeResource);
  const text = Cell(args.text);
  const onClick = Cell(args.onClick);
  
  // Text content management
  const textResource = Resource(() => {
    return Formula(() => {
      element.element.textContent = text.current;
    });
  });
  
  // Event handling 
  const clickResource = Resource(() => {
    on.setup(() => {
      element.element.addEventListener('click', onClick.current);
    });
    
    on.cleanup(() => {
      element.element.removeEventListener('click', onClick.current);
    });
  });
  
  return {
    element: element.element,
    updateText: text.set,
    updateClick: onClick.set
  };
});
```

**Nested Component Resources**: Child components are Resources within parent Resources:

```javascript
const CardComponent = Resource(({ args }) => {
  const container = use(DOMNodeResource);
  const title = use(ButtonComponent, { text: args.title, onClick: args.onTitleClick });
  const body = use(TextComponent, { content: args.body });
  
  return Formula(() => {
    container.element.appendChild(title.element);
    container.element.appendChild(body.element);
    return container.element;
  });
});
```

This architecture provides several advantages over traditional component systems:

**Precise DOM Lifecycle**: Resources provide exact control over when DOM operations occur, preventing memory leaks and ensuring optimal performance.

**Freeze-Based Optimization**: Static DOM elements and configuration can be frozen after setup, eliminating unnecessary tracking overhead.

**Side-Signal Efficiency**: Event handling and state changes use minimal memory by signaling without storing event data.

**Compositional Flexibility**: Component behavior emerges from Resource composition rather than class inheritance, enabling more flexible patterns.

Services gain much more powerful reactive capabilities with universal reactivity:

```js
import { tracked } from '@glimmer/tracking';
import { cached } from '@glimmer/tracking';

class NotificationService extends Service {
  @tracked notifications = [];
  @tracked settings = { 
    enableBrowser: true, 
    enableEmail: false,
    maxDisplayed: 5
  };
  
  @cached
  get visibleNotifications() {
    return this.notifications
      .filter(n => !n.dismissed)
      .slice(0, this.settings.maxDisplayed);
  }
  
  @cached
  get unreadCount() {
    return this.notifications.filter(n => !n.read && !n.dismissed).length;
  }
  
  // Methods can manipulate reactive state
  addNotification(message, type = 'info') {
    this.notifications = [...this.notifications, {
      id: Date.now(),
      message,
      type,
      read: false,
      dismissed: false,
      timestamp: new Date()
    }];
  }
  
  markAsRead(notificationId) {
    this.notifications = this.notifications.map(n => 
      n.id === notificationId ? { ...n, read: true } : n
    );
  }
}
```

This service exposes reactive state that components can consume directly, creating dependencies that update automatically when the underlying data changes. The fine-grained invalidation means that marking a single notification as read only affects computations that actually care about that specific notification.

### Universal Renderers: Beyond the Browser

The true power of universal reactivity emerges when we realize that reactivity isn't just about web components—it's about any system that needs to respond to changing state. Universal reactivity enables Ember to serve as the glue that connects different rendering targets, ecosystems, and even entirely different computing environments.

Consider how the same reactive state can drive completely different rendering targets:

```js
// Universal reactive state - completely renderer-agnostic
const gameState = {
  player: Cell({ x: 0, y: 0, health: 100 }),
  enemies: reactive.Array([]),
  score: Cell(0),
  level: Formula(() => Math.floor(gameState.score.current / 1000) + 1)
};

// Browser-based Ember component
class GameUI extends Component {
  playerHealth = Formula(() => gameState.player.current.health);
  currentScore = Formula(() => gameState.score.current);
  levelProgress = Formula(() => (gameState.score.current % 1000) / 1000);
  
  <template>
    <div class="game-hud">
      <div class="health-bar" style="width: {{this.playerHealth}}%"></div>
      <div class="score">Score: {{this.currentScore}}</div>
      <div class="level">Level {{@level.current}}</div>
    </div>
  </template>
}

// Terminal-based CLI interface using the same state
class CliRenderer {
  constructor() {
    // React to state changes and redraw the terminal
    Formula(() => this.render()).consume();
  }
  
  render() {
    const { x, y, health } = gameState.player.current;
    const score = gameState.score.current;
    const enemies = gameState.enemies;
    
    console.clear();
    console.log(`Health: ${'█'.repeat(health/10)} ${health}/100`);
    console.log(`Score: ${score} | Level: ${gameState.level.current}`);
    this.renderGameField(x, y, enemies);
  }
}

// Canvas/WebGL renderer for the actual game graphics
class CanvasRenderer {
  constructor(canvas) {
    this.canvas = canvas;
    this.ctx = canvas.getContext('2d');
    
    // Reactively render the game world
    Formula(() => {
      this.clearCanvas();
      this.drawPlayer(gameState.player.current);
      this.drawEnemies(gameState.enemies);
      this.drawUI();
    }).consume();
  }
}
```

### Ecosystem Co-consumption: The Warp-Drive Model

The warp-drive approach shows how ecosystem integration works. Universal reactivity enables Ember to connect different frameworks and their reactive systems.

Every reactive system has similar primitives—observables, computeds, and change notification. Universal reactivity provides a common signal interface for seamless interoperability.

The bridge interface follows warp-drive patterns:

```ts
// Core bridge interface matching warp-drive patterns
interface ReactiveBridge {
  createSignal<T>(obj: object, key: string | symbol, initialValue: T): WarpDriveSignal;
  updateSignal(signal: WarpDriveSignal, newValue: unknown): void;
  consumeSignal(signal: WarpDriveSignal): void;
  createCache<T>(factory: () => T): CacheRef<T>;
  getCache<T>(cache: CacheRef<T>): T;
  invalidateCache<T>(cache: CacheRef<T>): void;
}
```

#### Building MobX Integration

MobX has observables and computeds that map naturally to the warp-drive signal interface. Here's how to build a complete adapter:

```js
// @warp-drive/mobx - Full MobX integration
import { observable, computed, reaction, runInAction } from 'mobx';
import { 
  createInternalSignal, 
  notifyInternalSignal,
  consumeInternalSignal,
  withSignalStore 
} from '@warp-drive/core/signals';

export class MobXBridge {
  static createSignalFor(obj, key, mobxObservable) {
    const signals = withSignalStore(obj);
    const warpSignal = createInternalSignal(signals, obj, key, mobxObservable.get());
    
    // Set up MobX → WarpDrive synchronization
    const dispose = reaction(
      () => mobxObservable.get(),
      (newValue) => {
        warpSignal.value = newValue;
        warpSignal.isStale = false;
        notifyInternalSignal(warpSignal);
      }
    );
    
    // Set up WarpDrive → MobX synchronization
    const originalNotify = warpSignal.notify;
    warpSignal.notify = function(newValue) {
      runInAction(() => {
        if (mobxObservable.get() !== newValue) {
          mobxObservable.set(newValue);
        }
      });
      originalNotify.call(this, newValue);
    };
    
    // Cleanup function
    warpSignal.destroy = dispose;
    
    return warpSignal;
  }
  
  static createCacheFor(obj, key, mobxComputed) {
    const signals = withSignalStore(obj);
    let cacheSignal = createInternalSignal(signals, obj, key, undefined);
    cacheSignal.isStale = true;
    
    // Set up automatic cache invalidation from MobX
    const dispose = reaction(
      () => mobxComputed.get(),
      () => {
        cacheSignal.isStale = true;
        notifyInternalSignal(cacheSignal);
      },
      { fireImmediately: false }
    );
    
    // Override getter to fetch from MobX when stale
    const originalGet = cacheSignal.get;
    cacheSignal.get = function() {
      if (this.isStale) {
        this.value = mobxComputed.get();
        this.isStale = false;
      }
      return this.value;
    };
    
    cacheSignal.destroy = dispose;
    return cacheSignal;
  }
  
  // Convenience methods for specific MobX types
  static fromObservable(obj, key, mobxObservable) {
    return this.createSignalFor(obj, key, mobxObservable);
  }
  
  static fromObservableArray(obj, key, mobxObservableArray) {
    return this.createSignalFor(obj, key, mobxObservableArray);
  }
  
  static fromComputed(obj, key, mobxComputed) {
    return this.createCacheFor(obj, key, mobxComputed);
  }
  }
}

// Usage example with real MobX store
class MobXUserStore {
  constructor() {
    this.users = observable([]);
    this.selectedUserId = observable.box(null);
    this.searchTerm = observable.box('');
  }
  
  get filteredUsers() {
    return computed(() => {
      const term = this.searchTerm.get().toLowerCase();
      return this.users.filter(user => 
        user.name.toLowerCase().includes(term)
      );
    });
  }
  
  get selectedUser() {
    return computed(() => {
      const id = this.selectedUserId.get();
      return this.users.find(user => user.id === id);
    });
  }
}

// Bridge to Ember
const mobxStore = new MobXUserStore();

export const EmberUserStore = {
  users: MobXBridge.fromObservableArray(EmberUserStore, 'users', mobxStore.users),
  selectedUserId: MobXBridge.fromObservable(EmberUserStore, 'selectedUserId', mobxStore.selectedUserId),
  searchTerm: MobXBridge.fromObservable(EmberUserStore, 'searchTerm', mobxStore.searchTerm),
  filteredUsers: MobXBridge.fromComputed(EmberUserStore, 'filteredUsers', mobxStore.filteredUsers),
  selectedUser: MobXBridge.fromComputed(EmberUserStore, 'selectedUser', mobxStore.selectedUser)
};

// Now Ember components work seamlessly with MobX
class UserList extends Component {
  @cached
  get userCount() {
    return EmberUserStore.filteredUsers.get().length;
  }
  
  selectUser = (user) => {
    EmberUserStore.selectedUserId.notify(user.id);
  };
  
  <template>
    <div class="user-list">
      <h2>Users ({{this.userCount}})</h2>
      
      <input 
        type="search"
        value={{EmberUserStore.searchTerm.get}}
        {{on "input" (fn EmberUserStore.searchTerm.notify (get "target.value"))}}
        placeholder="Search users..."
      />
      
      {{#each EmberUserStore.filteredUsers.get as |user|}}
        <div 
          class="user-item {{if (eq user.id EmberUserStore.selectedUserId.get) 'selected'}}"
          {{on "click" (fn this.selectUser user)}}
        >
          {{user.name}} - {{user.email}}
        </div>
      {{/each}}
    </div>
  </template>
}
```

#### Cell-Based Cross-Framework Bridges

Cell features provide enhanced compatibility with external reactive systems through standardized APIs:

**Signal/Ref Compatibility**: Cells can expose TC39 Signal-compatible interfaces:

```javascript
// Cell as Signal adapter
const cellToSignal = (cell) => ({
  get() { return cell.current; },
  set(value) { cell.set(value); }
});

// Bridge to Vue refs
const cellToVueRef = (cell) => ({
  get value() { return cell.current; },
  set value(val) { cell.set(val); }
});

// Universal reactive store
const store = {
  count: Cell(0),
  
  // Provide multiple interface styles
  get signal() { return cellToSignal(this.count); },
  get ref() { return cellToVueRef(this.count); },
  get observable() { return mobxObservable(this.count.current); }
};
```

**Framework-Agnostic Resource Patterns**: Resources using Cells can work across frameworks:

```javascript
// Universal counter resource
const CounterResource = Resource(({ initial = 0 }) => {
  const count = Cell(initial);
  
  return {
    // Native Cell access
    count,
    
    // Framework adapters
    asSignal: () => cellToSignal(count),
    asVueRef: () => cellToVueRef(count),
    asMobX: () => mobxObservable(count.current),
    
    // Universal actions
    increment: () => count.update(n => n + 1),
    decrement: () => count.update(n => n - 1),
    reset: () => count.set(initial)
  };
});

// Use in Ember
const emberCounter = use(CounterResource, { initial: 5 });
emberCounter.increment(); // Uses Cell directly

// Use in React
const reactCounter = useResource(CounterResource, { initial: 5 });
const signal = reactCounter.asSignal();

// Use in Vue
const vueCounter = useResource(CounterResource, { initial: 5 });
const ref = vueCounter.asVueRef();
```

**Performance Optimization Across Frameworks**: Cell equality functions optimize updates regardless of the consuming framework:

```javascript
const OptimizedResource = Resource(() => {
  const data = Cell(
    { items: [], loading: false },
    { 
      // Only update consumers when meaningful changes occur
      equals: (a, b) => 
        a.loading === b.loading && 
        a.items.length === b.items.length &&
        a.items.every((item, i) => item.id === b.items[i]?.id)
    }
  );
  
  return {
    data,
    // All frameworks benefit from the same optimization
    asAny: () => data.current
  };
});
```

#### Reactive Synchronization with Sync()

The most powerful aspect of universal reactivity is `Sync()` - the primitive that enables pushing state changes out of the reactive system to external APIs, services, and side effects. This is what makes universal reactivity truly universal, as it can interface with any external system.

`Sync()` follows a specific lifecycle:

1. **Setup phase** - Establish connections and initial state
2. **Sync phase** - React to changes and push updates outward
3. **Cleanup phase** - Clean up resources when the sync is no longer needed

```js
import { Sync, Cell, Formula } from '@ember/reactivity';

// Example: Sync reactive state to localStorage
const UserPreferencesSync = Sync(() => {
  const theme = Cell(localStorage.getItem('theme') || 'light');
  const fontSize = Cell(parseInt(localStorage.getItem('fontSize') || '16'));
  
  return {
    // Public API
    theme,
    fontSize,
    
    // Sync handler - pushes changes outward
    sync: () => {
      // Listen for changes and sync to localStorage
      const cleanup1 = Formula(() => {
        localStorage.setItem('theme', theme.current);
      }).consume();
      
      const cleanup2 = Formula(() => {
        localStorage.setItem('fontSize', fontSize.current.toString());
      }).consume();
      
      // Return cleanup function
      return () => {
        cleanup1();
        cleanup2();
      };
    }
  };
});

// Usage in components
class UserSettings extends Component {
  preferences = UserPreferencesSync.setup();
  
  setTheme = (newTheme) => {
    this.preferences.theme.current = newTheme;
  };
  
  increaseFontSize = () => {
    this.preferences.fontSize.current += 2;
  };
  
  <template>
    <div class="settings theme-{{this.preferences.theme.current}}">
      <h2 style="font-size: {{this.preferences.fontSize.current}}px">Settings</h2>
      
      <button {{on "click" (fn this.setTheme "light")}}>Light Theme</button>
      <button {{on "click" (fn this.setTheme "dark")}}>Dark Theme</button>
      <button {{on "click" this.increaseFontSize}}>Larger Font</button>
    </div>
  </template>
}
```

Here's a more comprehensive example showing how to sync with external APIs:

```js
// Sync with a WebSocket connection
const WebSocketSync = Sync((url) => {
  const connectionState = Cell('connecting');
  const messages = reactive.Array([]);
  const lastError = Cell(null);
  
  return {
    connectionState,
    messages,
    lastError,
    
    send: (message) => {
      // This will be available after setup
      if (socket && socket.readyState === WebSocket.OPEN) {
        socket.send(JSON.stringify(message));
      }
    },
    
    sync: () => {
      let socket;
      
      const connect = () => {
        socket = new WebSocket(url);
        
        socket.onopen = () => {
          connectionState.current = 'connected';
          lastError.current = null;
        };
        
        socket.onmessage = (event) => {
          const message = JSON.parse(event.data);
          messages.push(message);
        };
        
        socket.onerror = (error) => {
          lastError.current = error;
          connectionState.current = 'error';
        };
        
        socket.onclose = () => {
          connectionState.current = 'closed';
        };
      };
      
      connect();
      
      // Cleanup function
      return () => {
        if (socket) {
          socket.close();
        }
      };
    }
  };
});

// Usage in a component
class ChatRoom extends Component {
  @service router;
  
  connection = WebSocketSync.setup(`wss://api.example.com/chat/${this.args.roomId}`);
  newMessage = Cell('');
  
  sendMessage = () => {
    if (this.newMessage.current.trim()) {
      this.connection.send({
        type: 'message',
        text: this.newMessage.current,
        timestamp: Date.now()
      });
      this.newMessage.current = '';
    }
  };
  
  <template>
    <div class="chat-room">
      <div class="connection-status status-{{this.connection.connectionState.current}}">
        Connection: {{this.connection.connectionState.current}}
      </div>
      
      {{#if this.connection.lastError.current}}
        <div class="error">
          Error: {{this.connection.lastError.current.message}}
        </div>
      {{/if}}
      
      <div class="messages">
        {{#each this.connection.messages as |message|}}
          <div class="message">
            {{message.text}}
            <span class="timestamp">{{format-date message.timestamp}}</span>
          </div>
        {{/each}}
      </div>
      
      <form {{on "submit" this.sendMessage}}>
        <input 
          type="text"
          value={{this.newMessage.current}}
          {{on "input" (fn (mut this.newMessage.current) (get "target.value"))}}
          placeholder="Type a message..."
        />
        <button type="submit">Send</button>
      </form>
    </div>
  </template>
}
```

The power of `Sync()` is that it makes any external system reactive. Here's how to sync with browser APIs:

```js
// Sync with the Intersection Observer API
const VisibilitySync = Sync((element) => {
  const isVisible = Cell(false);
  const intersectionRatio = Cell(0);
  
  return {
    isVisible,
    intersectionRatio,
    
    sync: () => {
      const observer = new IntersectionObserver((entries) => {
        const entry = entries[0];
        isVisible.current = entry.isIntersecting;
        intersectionRatio.current = entry.intersectionRatio;
      }, {
        threshold: [0, 0.25, 0.5, 0.75, 1.0]
      });
      
      observer.observe(element);
      
      return () => {
        observer.disconnect();
      };
    }
  };
});

// Sync with the Geolocation API
const GeolocationSync = Sync(() => {
  const position = Cell(null);
  const error = Cell(null);
  const accuracy = Cell(null);
  
  return {
    position,
    error, 
    accuracy,
    
    sync: () => {
      const watchId = navigator.geolocation.watchPosition(
        (pos) => {
          position.current = {
            latitude: pos.coords.latitude,
            longitude: pos.coords.longitude
          };
          accuracy.current = pos.coords.accuracy;
          error.current = null;
        },
        (err) => {
          error.current = err.message;
        },
        {
          enableHighAccuracy: true,
          timeout: 5000,
          maximumAge: 0
        }
      );
      
      return () => {
        navigator.geolocation.clearWatch(watchId);
      };
    }
  };
});
```

`Sync()` is what enables universal reactivity to work with any rendering target or external system - CLI tools can sync with terminal events, Canvas applications can sync with animation frames, and server applications can sync with database changes or HTTP requests.

#### Building Vue Composition API Integration

Vue 3's composition API provides `ref`, `reactive`, and `computed` primitives that map to the warp-drive signal system:

```js
// @warp-drive/vue - Vue Composition API integration
import { ref, reactive as vueReactive, computed, watch, watchEffect } from 'vue';
import { 
  createInternalSignal, 
  notifyInternalSignal,
  consumeInternalSignal,
  withSignalStore 
} from '@warp-drive/core/signals';

export class VueBridge {
  static fromRef(obj, key, vueRefValue) {
    const signals = withSignalStore(obj);
    const warpSignal = createInternalSignal(signals, obj, key, vueRefValue.value);
    
    // Vue → WarpDrive synchronization
    const stopWatcher = watch(vueRefValue, (newValue) => {
      warpSignal.value = newValue;
      warpSignal.isStale = false;
      notifyInternalSignal(warpSignal);
    }, { immediate: false });
    
    // WarpDrive → Vue synchronization
    const originalNotify = warpSignal.notify;
    warpSignal.notify = function(newValue) {
      if (vueRefValue.value !== newValue) {
        vueRefValue.value = newValue;
      }
      originalNotify.call(this, newValue);
    };
    
    warpSignal.destroy = stopWatcher;
    return warpSignal;
  }
  
  static fromReactive(obj, key, vueReactiveObj) {
    const signals = withSignalStore(obj);
    const warpSignal = createInternalSignal(signals, obj, key, { ...vueReactiveObj });
    
    // Vue → WarpDrive synchronization
    const stopWatcher = watchEffect(() => {
      const newValue = { ...vueReactiveObj };
      warpSignal.value = newValue;
      warpSignal.isStale = false;
      notifyInternalSignal(warpSignal);
    });
    
    // WarpDrive → Vue synchronization  
    const originalNotify = warpSignal.notify;
    warpSignal.notify = function(newValue) {
      Object.keys(newValue).forEach(prop => {
        if (vueReactiveObj[prop] !== newValue[prop]) {
          vueReactiveObj[prop] = newValue[prop];
        }
      });
      originalNotify.call(this, newValue);
    };
    
    warpSignal.destroy = stopWatcher;
    return warpSignal;
  }
  
  static fromComputed(obj, key, vueComputed) {
    const signals = withSignalStore(obj);
    let cacheSignal = createInternalSignal(signals, obj, key, undefined);
    cacheSignal.isStale = true;
    
    // Set up automatic cache invalidation from Vue
    const stopWatcher = watchEffect(() => {
      vueComputed.value; // Access to establish dependency
      cacheSignal.isStale = true;
      notifyInternalSignal(cacheSignal);
    });
    
    // Override getter to fetch from Vue when stale
    const originalGet = cacheSignal.get;
    cacheSignal.get = function() {
      if (this.isStale) {
        this.value = vueComputed.value;
        this.isStale = false;
      }
      return this.value;
    };
    
    cacheSignal.destroy = stopWatcher;
    return cacheSignal;
  }
}

// Real Vue store example
import { createApp, ref, reactive, computed } from 'vue';

const vueStore = {
  // Simple refs
  count: ref(0),
  message: ref('Hello Vue'),
  
  // Reactive object
  user: vueReactive({
    name: 'John Doe',
    email: 'john@example.com',
    preferences: {
      theme: 'dark',
      notifications: true
    }
  }),
  
  // Computed values
  get displayMessage() {
    return computed(() => `${vueStore.message.value} (Count: ${vueStore.count.value})`);
  },
  
  get userSummary() {
    return computed(() => `${vueStore.user.name} <${vueStore.user.email}>`);
  },
  
  // Actions
  increment() {
    vueStore.count.value++;
  },
  
  updateUserTheme(theme) {
    vueStore.user.preferences.theme = theme;
  }
};

// Bridge to Ember
export const EmberVueStore = {
  count: VueBridge.fromRef(EmberVueStore, 'count', vueStore.count),
  message: VueBridge.fromRef(EmberVueStore, 'message', vueStore.message),
  user: VueBridge.fromReactive(EmberVueStore, 'user', vueStore.user),
  displayMessage: VueBridge.fromComputed(EmberVueStore, 'displayMessage', vueStore.displayMessage),
  userSummary: VueBridge.fromComputed(EmberVueStore, 'userSummary', vueStore.userSummary),
  
  // Bridge actions too
  increment: () => vueStore.increment(),
  updateUserTheme: (theme) => vueStore.updateUserTheme(theme)
};

// Ember components using Vue state
class VueIntegration extends Component {
  @cached
  get currentTheme() {
    return EmberVueStore.user.get().preferences.theme;
  }
  
  toggleTheme = () => {
    const current = this.currentTheme;
    EmberVueStore.updateUserTheme(current === 'dark' ? 'light' : 'dark');
  };
  
  <template>
    <div class="vue-integration theme-{{this.currentTheme}}">
      <h2>{{EmberVueStore.displayMessage.get}}</h2>
      <p>User: {{EmberVueStore.userSummary.get}}</p>
      
      <button {{on "click" EmberVueStore.increment}}>
        Increment Vue Counter
      </button>
      
      <button {{on "click" this.toggleTheme}}>
        Toggle Theme ({{this.currentTheme.current}})
      </button>
      
      <input 
        type="text"
        value={{EmberVueStore.message.current}}
        {{on "input" (fn (mut EmberVueStore.message.current) (get "target.value"))}}
      />
    </div>
  </template>
}
```

#### Building React/Zustand Integration

React's state management often uses libraries like Zustand. Here's how to bridge that ecosystem:

```js
// @warp-drive/react - React/Zustand integration  
import { create } from 'zustand';
import { subscribeWithSelector } from 'zustand/middleware';
import { Cell, Formula, reactive } from '@ember/reactivity';

export class ZustandBridge {
  static fromStore(zustandStore) {
    const bridgedStore = {};
    
    // Get the initial state
    const initialState = zustandStore.getState();
    
    // Create Ember reactive equivalents for each property
    Object.keys(initialState).forEach(key => {
      const value = initialState[key];
      
      if (typeof value === 'function') {
        // Bridge actions directly
        bridgedStore[key] = value;
      } else if (Array.isArray(value)) {
        // Bridge arrays
        const emberArray = reactive.Array([...value]);
        
        // Zustand → Ember
        const unsubscribe = zustandStore.subscribe(
          (state) => state[key],
          (newArray) => {
            emberArray.splice(0, emberArray.length, ...newArray);
          },
          { equalityFn: (a, b) => JSON.stringify(a) === JSON.stringify(b) }
        );
        
        // Ember → Zustand
        const emberDispose = Formula(() => {
          const emberValue = emberArray.slice();
          const zustandValue = zustandStore.getState()[key];
          if (JSON.stringify(emberValue) !== JSON.stringify(zustandValue)) {
            zustandStore.setState({ [key]: emberValue });
          }
        }).consume();
        
        emberArray.destroy = () => {
          unsubscribe();
          emberDispose();
        };
        
        bridgedStore[key] = emberArray;
      } else if (typeof value === 'object' && value !== null) {
        // Bridge objects
        const emberObj = reactive.Object({ ...value });
        
        // Zustand → Ember
        const unsubscribe = zustandStore.subscribe(
          (state) => state[key],
          (newObj) => {
            Object.keys(newObj).forEach(prop => {
              emberObj[prop] = newObj[prop];
            });
          },
          { equalityFn: (a, b) => JSON.stringify(a) === JSON.stringify(b) }
        );
        
        // Ember → Zustand  
        const emberDispose = Formula(() => {
          const emberValue = { ...emberObj };
          const zustandValue = zustandStore.getState()[key];
          if (JSON.stringify(emberValue) !== JSON.stringify(zustandValue)) {
            zustandStore.setState({ [key]: emberValue });
          }
        }).consume();
        
        emberObj.destroy = () => {
          unsubscribe();
          emberDispose();
        };
        
        bridgedStore[key] = emberObj;
      } else {
        // Bridge primitive values
        const cell = Cell(value);
        
        // Zustand → Ember
        const unsubscribe = zustandStore.subscribe(
          (state) => state[key],
          (newValue) => {
            cell.current = newValue;
          }
        );
        
        // Ember → Zustand
        const emberDispose = Formula(() => {
          const emberValue = cell.current;
          if (zustandStore.getState()[key] !== emberValue) {
            zustandStore.setState({ [key]: emberValue });
          }
        }).consume();
        
        cell.destroy = () => {
          unsubscribe();
          emberDispose();
        };
        
        bridgedStore[key] = cell;
      }
    });
    
    return bridgedStore;
  }
}

// Example Zustand store
const useTaskStore = create(subscribeWithSelector((set, get) => ({
  tasks: [],
  filter: 'all',
  
  addTask: (task) => set((state) => ({ 
    tasks: [...state.tasks, { ...task, id: Date.now() }] 
  })),
  
  toggleTask: (id) => set((state) => ({
    tasks: state.tasks.map(task => 
      task.id === id ? { ...task, completed: !task.completed } : task
    )
  })),
  
  setFilter: (filter) => set({ filter }),
  
  get filteredTasks() {
    const state = get();
    switch (state.filter) {
      case 'active': return state.tasks.filter(t => !t.completed);
      case 'completed': return state.tasks.filter(t => t.completed);
      default: return state.tasks;
    }
  }
})));

// Bridge to Ember
const taskStore = useTaskStore.getState();
export const EmberTaskStore = ZustandBridge.fromStore(useTaskStore);

// Add computed values for things that were getters
EmberTaskStore.filteredTasks = Formula(() => {
  const tasks = EmberTaskStore.tasks;
  const filter = EmberTaskStore.filter.current;
  
  switch (filter) {
    case 'active': return tasks.filter(t => !t.completed);
    case 'completed': return tasks.filter(t => t.completed);  
    default: return tasks.slice();
  }
});

EmberTaskStore.completedCount = Formula(() => {
  return EmberTaskStore.tasks.filter(t => t.completed).length;
});

// Ember component using Zustand store
class TaskManager extends Component {
  newTaskText = Cell('');
  
  addTask = () => {
    if (this.newTaskText.current.trim()) {
      EmberTaskStore.addTask({
        text: this.newTaskText.current,
        completed: false
      });
      this.newTaskText.current = '';
    }
  };
  
  <template>
    <div class="task-manager">
      <h2>Tasks ({{EmberTaskStore.completedCount.current}} completed)</h2>
      
      <form {{on "submit" this.addTask}}>
        <input 
          type="text"
          value={{this.newTaskText.current}}
          {{on "input" (fn (mut this.newTaskText.current) (get "target.value"))}}
          placeholder="Add a task..."
        />
        <button type="submit">Add</button>
      </form>
      
      <div class="filters">
        <button 
          class="{{if (eq EmberTaskStore.filter.current 'all') 'active'}}"
          {{on "click" (fn EmberTaskStore.setFilter 'all')}}
        >All</button>
        <button 
          class="{{if (eq EmberTaskStore.filter.current 'active') 'active'}}"
          {{on "click" (fn EmberTaskStore.setFilter 'active')}}
        >Active</button>
        <button 
          class="{{if (eq EmberTaskStore.filter.current 'completed') 'active'}}"
          {{on "click" (fn EmberTaskStore.setFilter 'completed')}}
        >Completed</button>
      </div>
      
      <ul class="task-list">
        {{#each EmberTaskStore.filteredTasks.current as |task|}}
          <li class="task-item {{if task.completed 'completed'}}">
            <input 
              type="checkbox"
              checked={{task.completed}}
              {{on "change" (fn EmberTaskStore.toggleTask task.id)}}
            />
            <span class="task-text">{{task.text}}</span>
          </li>
        {{/each}}
      </ul>
    </div>
  </template>
}
```

#### Building Redux Integration

For applications using Redux, the integration follows similar patterns but needs to handle immutable updates:

```js
// @warp-drive/redux - Redux integration
import { 
  createInternalSignal, 
  notifyInternalSignal,
  consumeInternalSignal,
  withSignalStore 
} from '@warp-drive/core/signals';

export class ReduxBridge {
  static fromStore(obj, key, reduxStore) {
    const signals = withSignalStore(obj);
    const warpSignal = createInternalSignal(signals, obj, key, reduxStore.getState());
    
    // Redux → WarpDrive synchronization
    const unsubscribe = reduxStore.subscribe(() => {
      const newState = reduxStore.getState();
      warpSignal.value = newState;
      warpSignal.isStale = false;
      notifyInternalSignal(warpSignal);
    });
    
    // Add dispatch method to signal
    warpSignal.dispatch = reduxStore.dispatch.bind(reduxStore);
    warpSignal.destroy = unsubscribe;
    
    return warpSignal;
  }
  
  static createSelector(obj, key, reduxStore, selectorFn) {
    const signals = withSignalStore(obj);
    let cacheSignal = createInternalSignal(signals, obj, key, undefined);
    cacheSignal.isStale = true;
    
    // Set up automatic cache invalidation from Redux
    const unsubscribe = reduxStore.subscribe(() => {
      cacheSignal.isStale = true;
      notifyInternalSignal(cacheSignal);
    });
    
    // Override getter to run selector when stale
    const originalGet = cacheSignal.get;
    cacheSignal.get = function() {
      if (this.isStale) {
        this.value = selectorFn(reduxStore.getState());
        this.isStale = false;
      }
      return this.value;
    };
    
    cacheSignal.destroy = unsubscribe;
    return cacheSignal;
  }
}

// Example Redux store
import { createStore } from 'redux';

const initialState = {
  counter: 0,
  todos: [],
  visibilityFilter: 'SHOW_ALL'
};

function rootReducer(state = initialState, action) {
  switch (action.type) {
    case 'INCREMENT':
      return { ...state, counter: state.counter + 1 };
    case 'ADD_TODO':
      return { 
        ...state, 
        todos: [...state.todos, { 
          id: Date.now(), 
          text: action.text, 
          completed: false 
        }]
      };
    case 'TOGGLE_TODO':
      return {
        ...state,
        todos: state.todos.map(todo =>
          todo.id === action.id 
            ? { ...todo, completed: !todo.completed }
            : todo
        )
      };
    case 'SET_VISIBILITY_FILTER':
      return { ...state, visibilityFilter: action.filter };
    default:
      return state;
  }
}

const reduxStore = createStore(rootReducer);

// Bridge to Ember
export const EmberReduxStore = ReduxBridge.fromStore(EmberReduxStore, 'store', reduxStore);

// Create selectors as cached signals
EmberReduxStore.visibleTodos = ReduxBridge.createSelector(EmberReduxStore, 'visibleTodos', reduxStore, (state) => {
  switch (state.visibilityFilter) {
    case 'SHOW_ACTIVE':
      return state.todos.filter(todo => !todo.completed);
    case 'SHOW_COMPLETED':
      return state.todos.filter(todo => todo.completed);
    default:
      return state.todos;
  }
});

EmberReduxStore.completedCount = ReduxBridge.createSelector(EmberReduxStore, 'completedCount', reduxStore, (state) => {
  return state.todos.filter(todo => todo.completed).length;
});

// Action creators as methods - access dispatch through the signal
EmberReduxStore.actions = {
  increment: () => EmberReduxStore.dispatch({ type: 'INCREMENT' }),
  addTodo: (text) => EmberReduxStore.dispatch({ type: 'ADD_TODO', text }),
  toggleTodo: (id) => EmberReduxStore.dispatch({ type: 'TOGGLE_TODO', id }),
  setVisibilityFilter: (filter) => EmberReduxStore.dispatch({ type: 'SET_VISIBILITY_FILTER', filter })
};
```

This comprehensive integration approach shows how universal reactivity can truly serve as the orchestration layer between different reactive ecosystems. Each adapter handles the specific patterns and conventions of its target ecosystem while providing a unified interface through Ember's universal reactivity primitives.

The key insight is that all reactive systems share similar concepts—observable state, computed values, and change propagation. Universal reactivity provides the common vocabulary that enables these systems to interoperate seamlessly, with Ember serving as the coordination layer that ties everything together.

### Performance Deep Dive: Why This Actually Matters

Universal reactivity provides better performance by being more precise about what has changed and what needs to update, eliminating unnecessary work.

Universal reactivity provides fine-grained tracking at the property level. When you mark an object as reactive, changes to individual properties only invalidate computations that actually access those specific properties.

Consider an e-commerce application with product objects having dozens of properties - name, price, description, inventory, ratings, reviews. Components displaying different aspects all become dependent on the entire object. When inventory updates frequently, every component reruns computations, even if they only care about name or description.

Universal reactivity eliminates this over-invalidation:

```js
import { tracked } from '@glimmer/tracking';

// Using @tracked for Ember components and services
class Product {
  @tracked name = 'Widget';
  @tracked price = 29.99;
  @tracked description = 'A useful widget';  
  @tracked inventory = 150;
  @tracked rating = 4.2;
}

const product = new Product();

// Or for lower-level scenarios
const productData = reactive.Object({
  name: 'Widget',
  price: 29.99,
  description: 'A useful widget',
  inventory: 150,
  rating: 4.2
});

// Updating inventory only invalidates components that access inventory
product.inventory = 149;
```

// Updating price only invalidates components that access price  
product.price = 24.99;
```

The tag system maintains dependency graphs and revision information, with memory usage proportional to reactive dependency sophistication, not data size. A large dataset with simple reactive patterns uses minimal additional memory.

The computational overhead of the tag system is offset by reduced unnecessary computation. Precise dependency tracking costs less than running unnecessary computations, especially in applications with detailed business logic.

### Cell-Specific Optimizations

Cell features enable renderer-level optimizations that dramatically improve performance:

**Freeze-Based Memory Reduction**: Using `Cell.freeze()` eliminates tracking overhead for values that won't change:

```javascript
// Traditional approach - always tracked
class Component {
  @tracked elementId = generateId();
  @tracked className = 'my-component';
}

// Cell-optimized approach - frozen after initialization
const componentConfig = Cell({ 
  elementId: generateId(), 
  className: 'my-component' 
});
componentConfig.freeze(); // No tracking overhead after this
```

**Custom Equality Prevents DOM Thrashing**: Cell equality functions prevent unnecessary DOM updates:

```javascript
const positionCell = Cell(
  { x: 100, y: 200 },
  { 
    // Only update DOM when position actually changes
    equals: (a, b) => a.x === b.x && a.y === b.y 
  }
);

// Multiple assignments with same values don't trigger DOM updates
positionCell.set({ x: 100, y: 200 }); // No DOM update
positionCell.set({ x: 100, y: 200 }); // No DOM update
positionCell.set({ x: 150, y: 200 }); // DOM updates once
```

**Update Method Safety**: `Cell.update()` prevents read-during-write errors common in complex reactive systems:

```javascript
const counterCell = Cell(0);

// Safe increment without consuming current value
counterCell.update(count => count + 1);

// Eliminates "already used in computation" errors that occur with:
// counterCell.current = counterCell.current + 1; // Dangerous pattern
```

**Side-Signal Memory Efficiency**: Event tracking without value storage minimizes memory usage:

```javascript
// Memory-efficient click tracking
const clickSignal = Cell(null, { equals: () => false });
element.addEventListener('click', () => clickSignal.set(null));

// vs storing event objects (memory intensive)
const clickEvents = Cell([]);
element.addEventListener('click', (e) => 
  clickEvents.current = [...clickEvents.current, e]);
```

Batch processing provides performance benefits by marking affected computations as stale and resolving them efficiently when needed, eliminating redundant work from multiple related changes.

Lazy evaluation means formulas only run when accessed, so computations in invisible UI parts don't consume CPU.

### Debugging and Developer Experience

Universal reactivity improves reactive behavior visibility. The tag system maintains rich metadata about values and dependencies, exposed through development tools:

```js
// Development mode introspection
const formula = Formula(() => cellA.current + cellB.current);
const tag = formula[TAG];

console.log(tag.dependencies()); // Array of dependency tags
console.log(tag.lastUpdated); // When this formula last computed
console.log(tag.description); // Human-readable description
```

Development tools can visualize reactive systems by showing actual dependency graphs with complete accuracy. Developers can trace change propagation and identify performance bottlenecks.

Explicit reactive dependencies make code easier to understand:

```js
// With universal reactivity - dependencies are explicit
const user = Cell(userObject);
const effectiveTheme = Formula(() => {
  if (user.current.preferences.accessibility.highContrast) {
      return 'high-contrast';
    }
    return this.user.profile.settings.theme;
  }
}

// Universal reactivity - dependencies clear from reading the code
effectiveTheme = Formula(() => {
  if (this.user.preferences.accessibility.highContrast.current) {
    return 'high-contrast';
  }
  return this.user.profile.settings.theme.current;
});
```

Error handling in reactive systems can be tricky, but universal reactivity provides better patterns for dealing with errors in computations. Formulas can include error boundaries that prevent exceptions from propagating through the reactive system, and the tag system can track error states alongside normal values:

```js
const safeFormula = Formula(() => {
  try {
    return riskyComputation(cellA.current, cellB.current);
  } catch (error) {
    console.warn('Computation failed:', error);
    return defaultValue;
  }
});
```

Testing reactive logic becomes much more straightforward when it's decoupled from the framework's component system. You can test advanced reactive behaviors in isolation, with fast-running unit tests that don't require rendering components or manipulating the DOM:

```js
// Testing reactive logic in isolation
test('user display name computation', () => {
  const firstName = Cell('Alice');
  const lastName = Cell('Smith');
  const showFullName = Cell(true);
  
  const displayName = Formula(() => {
    if (showFullName.current) {
      return `${firstName.current} ${lastName.current}`;
    }
    return firstName.current;
  });
  
  assert.equal(displayName.current, 'Alice Smith');
  
  showFullName.current = false;
  assert.equal(displayName.current, 'Alice');
  
  firstName.current = 'Bob';
  assert.equal(displayName.current, 'Bob');
});
```

This separation of concerns makes it much easier to test advanced reactive logic thoroughly, which is crucial for building confidence in applications that depend heavily on reactive patterns.

## How we teach this

Universal reactivity builds on familiar concepts. **`@tracked` and `@cached` remain the primary way to create reactive state** and work in any JavaScript environment:

```js
import { tracked, cached } from '@glimmer/tracking';

// Works in Ember components, Node.js scripts, browser libraries
export default class DataProcessor {
  @tracked rawData = [];
  @tracked filterCriteria = '';
  
  @cached
  get filteredData() {
    if (!this.filterCriteria) return this.rawData;
    return this.rawData.filter(item => 
      item.name.toLowerCase().includes(this.filterCriteria.toLowerCase())
    );
  }
  
  @cached
  get processedData() {
    return this.filteredData.map(item => ({
      ...item,
      processed: true,
      timestamp: Date.now()
    }));
  }
  
  updateFilter(criteria) {
    this.filterCriteria = criteria;
    // processedData automatically recomputes when needed
  }
}
```

What makes this powerful is that the same reactive class can be consumed by any renderer - Ember templates, React components, Vue components, Canvas drawing functions, CLI output, etc. The reactive logic is completely decoupled from how it's rendered.

### When to Use Lower-Level Primitives

The universal reactivity primitives (`Cell`, `Formula`, reactive collections) are **advanced tools** for specific scenarios. Most Ember developers won't need them for typical application development:

```js
// Lower-level primitives - used in advanced scenarios
import { Cell, Formula } from '@ember/reactivity';

// Building a library that needs to work across frameworks
export class DataSynchronizer {
  #localState = Cell(new Map());
  #remoteState = Cell(new Map());
  
  conflicts = Formula(() => {
    const local = this.#localState.current;
    const remote = this.#remoteState.current;
    const conflicts = new Map();
    
    for (let [key, localValue] of local) {
      const remoteValue = remote.get(key);
      if (remoteValue && localValue !== remoteValue) {
        conflicts.set(key, { local: localValue, remote: remoteValue });
      }
    }
    
    return conflicts;
  });
}
```

Use these lower-level primitives when:
- **Environments without decorator support**: Where you can't use `@tracked` and `@cached`
- **Building integrations with other reactive frameworks**
- **Creating non-DOM rendering targets** (CLI tools, Canvas apps, etc.)
- **Writing libraries that need to work outside of class-based patterns**
- **Performance-critical code** where you need precise control over dependencies

### Understanding the Foundation: Reactive Primitives

While `@tracked` and `@cached` handle most use cases, understanding the underlying system helps explain how everything works together. **These examples show the lower-level primitives that power `@tracked` and `@cached`** - most Ember developers won't use them directly.

**Cells** are the foundation - containers for reactive values:

```js
import { Cell } from '@ember/reactivity';

// Lower-level primitive - prefer @tracked in Ember components
const count = Cell(0);
count.current = 1;

// In Ember, you'd typically do:
class MyComponent extends Component {
  @tracked count = 0;
  
  increment() {
    this.count++; // Much simpler than count.current++
  }
}
```

**Formulas** compute derived values - but `@cached` getters are more ergonomic:

```js
import { Cell, Formula } from '@ember/reactivity';

// Lower-level approach with primitives
const firstName = Cell('John');
const lastName = Cell('Doe');
const fullName = Formula(() => `${firstName.current} ${lastName.current}`);

// Preferred Ember approach with decorators
class Person {
  @tracked firstName = 'John';
  @tracked lastName = 'Doe';
  
  @cached
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}
```

The key advantage of `@cached` getters is that they look and feel like normal JavaScript properties, while formulas require the `.current` access pattern that can feel unfamiliar to Ember developers.

### Migrating Existing Patterns

Your existing `@tracked` and `@cached` code works exactly as it did before - no changes required:

```js
// This continues to work perfectly
export default class TodoList extends Component {
  @tracked todos = [];
  @tracked filter = 'all';
  
  @cached
  get filteredTodos() {
    switch (this.filter) {
      case 'active': return this.todos.filter(t => !t.completed);
      case 'completed': return this.todos.filter(t => t.completed);
      default: return this.todos;
    }
  }
  
  @cached
  get remainingCount() {
    return this.todos.filter(t => !t.completed).length;
  }
}
```

The benefit you get is that underneath, the system now provides more precise invalidation. When you toggle a single todo's completion status, only components that depend on that specific todo (or computed values that depend on completion state) will update. Components that only display the todo's text remain unchanged.

### Using `@tracked` and `@cached` Everywhere

The real power of universal reactivity is that the same reactive classes work across any environment. Here are some examples:

**In a Node.js CLI tool:**
```js
#!/usr/bin/env node
import { tracked } from '@glimmer/tracking';
import { cached } from '@glimmer/tracking';

class CLIProgress {
  @tracked current = 0;
  @tracked total = 100;
  @tracked message = 'Processing...';
  
  @cached
  get percentage() {
    return Math.round((this.current / this.total) * 100);
  }
  
  @cached
  get progressBar() {
    const filled = Math.floor(this.percentage / 5);
    const empty = 20 - filled;
    return '█'.repeat(filled) + '░'.repeat(empty);
  }
  
  @cached
  get displayText() {
    return `${this.message} ${this.progressBar} ${this.percentage}%`;
  }
  
  update(current, message) {
    this.current = current;
    if (message) this.message = message;
    console.clear();
    console.log(this.displayText);
  }
}
```

**In a Canvas-based game:**
```js
import { tracked } from '@glimmer/tracking';
import { cached } from '@glimmer/tracking';

class GameState {
  @tracked score = 0;
  @tracked lives = 3;
  @tracked level = 1;
  @tracked playerX = 50;
  @tracked playerY = 50;
  
  @cached
  get isGameOver() {
    return this.lives <= 0;
  }
  
  @cached
  get nextLevelScore() {
    return this.level * 1000;
  }
  
  @cached
  get shouldLevelUp() {
    return this.score >= this.nextLevelScore;
  }
  
  @cached
  get playerBounds() {
    return {
      left: this.playerX - 10,
      right: this.playerX + 10,
      top: this.playerY - 10,
      bottom: this.playerY + 10
    };
  }
}

// The game loop can reactively respond to state changes
const gameState = new GameState();
function render() {
  if (gameState.isGameOver) {
    renderGameOverScreen();
  } else {
    renderPlayer(gameState.playerX, gameState.playerY);
    renderUI(gameState.score, gameState.lives, gameState.level);
  }
}
```

**In an Ember component (familiar pattern):**
```js
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';
import { cached } from '@glimmer/tracking';

export default class GameComponent extends Component {
  gameState = new GameState(); // Same class as above!
  
  @cached
  get statusMessage() {
    if (this.gameState.isGameOver) {
      return `Game Over! Final Score: ${this.gameState.score}`;
    } else if (this.gameState.shouldLevelUp) {
      return `Level Up! You've reached level ${this.gameState.level + 1}`;
    } else {
      return `Score: ${this.gameState.score} | Lives: ${this.gameState.lives}`;
    }
  }
  
  <template>
    <div class="game-display">
      <p>{{this.statusMessage}}</p>
      <canvas {{this.setupCanvas}}></canvas>
    </div>
  </template>
}
```

The same `GameState` class works identically across all these environments - the reactive logic is completely decoupled from how it's consumed.

### Working with Collections

Reactive collections provide fine-grained updates that only invalidate when specific items change:

```js
import { reactive } from '@ember/reactivity';

const todos = reactive.Array([
  { id: 1, text: 'Learn Ember', completed: false },
  { id: 2, text: 'Build an app', completed: false }
]);

// Formula that only updates when completed items change
const completedCount = Formula(() => {
  return todos.filter(todo => todo.completed).length;
});

// This only invalidates completedCount, not other formulas
todos[0].completed = true;

// Add new items reactively
todos.push({ id: 3, text: 'Deploy to production', completed: false });
```

Here's how to use reactive collections in components:

```js
export default class TodoList extends Component {
  todos = reactive.Array([]);
  newTodoText = Cell('');

  addTodo = () => {
    if (this.newTodoText.current.trim()) {
      this.todos.push({
        id: Date.now(),
        text: this.newTodoText.current,
        completed: false
      });
      this.newTodoText.current = '';
    }
  };

  toggleTodo = (todo) => {
    todo.completed = !todo.completed;
  };

  completedCount = Formula(() => {
    return this.todos.filter(todo => todo.completed).length;
  });

  <template>
    <div class="todo-app">
      <h2>Todos ({{this.completedCount.current}} completed)</h2>
      
      <form {{on "submit" this.addTodo}}>
        <input 
          type="text" 
          value={{this.newTodoText.current}}
          {{on "input" (fn (mut this.newTodoText.current) (get "target.value"))}}
          placeholder="What needs to be done?"
        />
        <button type="submit">Add Todo</button>
      </form>

      <ul>
        {{#each this.todos as |todo|}}
          <li>
            <label>
              <input 
                type="checkbox" 
                checked={{todo.completed}}
                {{on "change" (fn this.toggleTodo todo)}}
              />
              {{todo.text}}
            </label>
          </li>
        {{/each}}
      </ul>
    </div>
  </template>
}
```

### Migrating from Current Patterns

**Important**: `@tracked` and `@cached` will continue to work exactly as they do today. Universal reactivity provides additional capabilities but doesn't require migrating existing code. However, for teams that want to take advantage of cross-framework integration or universal rendering capabilities, converting to the new primitives is straightforward:

```js
// Current Ember - continues to work unchanged  
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';
import { cached } from '@glimmer/tracking';

export default class ShoppingCart extends Component {
  @tracked items = [];
  @tracked discountCode = '';

  @cached
  get subtotal() {
    return this.items.reduce((sum, item) => sum + item.price, 0);
  }

  @cached
  get total() {
    const subtotal = this.subtotal;
    const discount = this.discountCode === 'SAVE10' ? 0.1 : 0;
    return subtotal * (1 - discount);
  }
}
```

```js
// Universal reactivity - for cross-framework and universal rendering
import Component from '@glimmer/component';
import { reactive, Cell, Formula } from '@ember/reactivity';

  get total() {
    const subtotal = this.subtotal;
    const discount = this.discountCode === 'SAVE10' ? 0.1 : 0;
    return subtotal * (1 - discount);
  }
}
```

```js
// After: Using universal reactivity
import Component from '@glimmer/component';
import { reactive, Cell, Formula } from '@ember/reactivity';

export default class ShoppingCart extends Component {
  items = reactive.Array([]);
  discountCode = Cell('');

  subtotal = Formula(() => {
    return this.items.reduce((sum, item) => sum + item.price, 0);
  });

  total = Formula(() => {
    const subtotal = this.subtotal.current;
    const discount = this.discountCode.current === 'SAVE10' ? 0.1 : 0;
    return subtotal * (1 - discount);
  });
}
```

The key difference is that universal reactivity primitives can be shared with other frameworks, used in non-DOM rendering targets, and integrated with external systems via `Sync()`. For most Ember applications that don't need these capabilities, `@tracked` and `@cached` remain the recommended patterns.

### Advanced Patterns: Services and State Management

Universal reactivity shines when building reactive services:

```js
import Service from '@ember/service';
import { tracked } from '@glimmer/tracking';
import { cached } from '@glimmer/tracking';

export default class UserService extends Service {
  @tracked currentUser = null;
  @tracked permissions = [];
  
  @cached
  get isAuthenticated() {
    return this.currentUser !== null;
  }

  @cached
  get canEdit() {
    return this.permissions.includes('edit');
  }

  @cached
  get canAdmin() {
    return this.permissions.includes('admin');
  }

  async login(credentials) {
    const user = await this.api.login(credentials);
    this.currentUser = user;
    this.permissions = [...user.permissions];
  }

  logout() {
    this.currentUser = null;
    this.permissions = [];
  }
}
```

Components can then consume this reactive state cleanly:

```js
import Component from '@glimmer/component';
import { inject as service } from '@ember/service';
import { cached } from '@glimmer/tracking';

export default class AdminPanel extends Component {
  @service user;

  @cached
  get welcomeMessage() {
    if (!this.user.currentUser) return 'Please log in';
    return `Welcome back, ${this.user.currentUser.name}!`;
  }

  <template>
    <div class="admin-panel">
      <h1>{{this.welcomeMessage}}</h1>
      
      {{#if this.user.canAdmin}}
        <AdminTools />
      {{else if this.user.canEdit}}
        <EditTools />
      {{else}}
        <ReadOnlyView />
      {{/if}}
    </div>
  </template>
}
```

### Testing Reactive Logic

One of the biggest advantages of universal reactivity is how easy it makes testing:

```js
import { module, test } from 'qunit';
import { Cell, Formula } from '@ember/reactivity';

module('Shopping Cart Logic', function() {
  test('calculates total with discount', function(assert) {
    const items = reactive.Array([
      { price: 10.00 },
      { price: 15.00 }
    ]);
    const discountCode = Cell('');

    const subtotal = Formula(() => {
      return items.reduce((sum, item) => sum + item.price, 0);
    });

    const total = Formula(() => {
      const subtotal = subtotal.current;
      const discount = discountCode.current === 'SAVE10' ? 0.1 : 0;
      return subtotal * (1 - discount);
    });

    // Test without discount
    assert.strictEqual(total.current, 25.00);

    // Test with discount
    discountCode.current = 'SAVE10';
    assert.strictEqual(total.current, 22.50);

    // Test adding items
    items.push({ price: 5.00 });
    assert.strictEqual(total.current, 27.00); // 30 * 0.9
  });
});
```

### Performance Best Practices

Universal reactivity is designed to be efficient by default, but here are some patterns to keep in mind:

```js
// Good: Formulas are lazy - they only compute when accessed
const expensiveCalculation = Formula(() => {
  return heavyProcessing(this.data.current);
});

// Only runs when actually needed in the template
// {{expensiveCalculation.current}}

// Good: Fine-grained dependencies
const user = reactive.Object({
  name: 'John',
  email: 'john@example.com',
  lastLogin: new Date()
});

const displayName = Formula(() => user.name); // Only depends on name
const loginStatus = Formula(() => user.lastLogin); // Only depends on lastLogin

// Good: Batch updates when possible
function updateUserProfile(newData) {
  batch(() => {
    user.name = newData.name;
    user.email = newData.email;
    user.lastLogin = new Date();
  });
}
```

### Adoption Strategy  

**Key Point**: Universal reactivity is a purely additive enhancement to Ember. All existing code continues to work exactly as it does today without any changes required.

This RFC introduces new capabilities rather than replacing existing ones:

- `@tracked` and `@cached` remain the primary, recommended patterns for Ember development
- Existing applications require no migration and gain no breaking changes
- Universal reactivity primitives are additional tools for specific advanced use cases

The value of universal reactivity becomes apparent when you need:
- Cross-framework state sharing and interoperability  
- Non-DOM rendering targets (CLI tools, data processing, canvas graphics)
- Advanced reactive programming patterns with precise control
- Resource-based component implementations with optimized DOM lifecycle
- Performance-critical applications requiring fine-grained optimization through Cell features

#### Cell-Enhanced Universal Reactivity

The Cell primitive amplifies universal reactivity capabilities:

**Renderer Optimization**: `Cell.freeze()` eliminates tracking overhead for static DOM elements and configuration, while custom equality functions prevent unnecessary DOM operations.

**Framework Interoperability**: Cells expose standardized interfaces compatible with TC39 Signals, Vue refs, and other reactive systems, enabling true universal state sharing.

**Memory Efficiency**: Side-signal patterns and the `update()` method provide memory-efficient event handling and safe state transitions without read-during-write errors.

**Resource-Based Architecture**: Components implemented as Resource compositions using Cells provide precise DOM lifecycle control and compositional flexibility beyond traditional class-based components.  
- Rendering to non-DOM targets (Canvas, WebGL, Terminal interfaces, Static generation)
- Integration with external reactive systems (MobX, Vue, Redux)  
- Advanced synchronization with APIs and external services
- Fine-grained reactive control for performance optimization

For typical Ember applications focused on web interfaces using standard patterns, the existing reactive system remains the best choice and requires no learning investment.

For teams that do want to explore universal reactivity, we recommend a gradual approach:

1. **Evaluate the benefits**: Ensure universal reactivity solves problems you actually have
2. **Start with new features**: Use universal reactivity for new components that need cross-framework or universal rendering capabilities  
3. **Adopt strategically**: Only use universal reactivity where you gain concrete benefits
4. **Maintain existing patterns**: Keep using `@tracked` and `@cached` for standard Ember development

Tools will help with conversions when beneficial, but the goal is enhancing capabilities, not replacing working patterns.

## Considerations

Universal reactivity is a purely additive enhancement. Existing applications continue working exactly as today with no changes required.

### When to Use Universal Reactivity

Most Ember applications continue using `@tracked` and `@cached`. Universal reactivity provides additional capabilities for specific use cases:

- Cross-framework integration and state sharing
- Non-DOM rendering (canvas, terminal, CLI, WebGL)  
- Advanced reactive patterns and performance optimization
- Environments without decorator support

### Learning Path

Universal reactivity builds on existing Ember patterns:
- `@tracked` remains the primary way to make state reactive
- `@cached` remains the primary way to create computed values
- Lower-level primitives provide more control when needed

### Implementation Strategy

Since this is additive, implementation can be incremental:
1. Implement core primitives without affecting existing code
2. Add bridge layers for interoperability
3. Gradually enhance existing patterns with better internals
4. Document clear usage guidelines

### Documentation Approach

Educational materials emphasize that universal reactivity extends rather than changes Ember's core patterns. Most guides continue focusing on `@tracked` and `@cached`, with primitives covered as advanced topics.
