# Snap Decorators Reference

```typescript
/**
 * TypeScript decorators for declarative event binding and dependency injection in Lens Studio.
 * Provides @bindStartEvent, @bindUpdateEvent, @bindDestroyEvent and others for clean event
 * handling, @memo for lazy-evaluated getters, @depends for component dependencies, and
 * @provider for parent component lookup with automatic error reporting.
 */

function bindEvent(eventType: keyof EventNameMap) {
  return function bindEventDecorator(target: Function, context: ClassMethodDecoratorContext) {
    const methodName = context.name;

    context.addInitializer(function(this: BaseScriptComponent) {
      const targetMethod = (this as any)[methodName] as Function;
      const innerAwake = (this as any)["onAwake"] as (() => void) | undefined;

      (this as any)["onAwake"] = function bindEventAwake() {
        this.createEvent(eventType).bind((...args: any[]) => {
          const result = targetMethod.call(this, ...args);
          if (result instanceof Promise) {
            return result.catch((err) => print("Error: " + err));
          }
          return result;
        });
        innerAwake?.call(this);
      };
    });
  };
}

export const bindStartEvent = bindEvent("OnStartEvent");
export const bindUpdateEvent = bindEvent("UpdateEvent");
export const bindDestroyEvent = bindEvent("OnDestroyEvent");

/**
 * Decorator to define a script component field as an injected dependency, auto-initialized with a reference to a
 * component found on a parent node.
 */
export function provider(componentType: keyof ComponentNameMap) {
  return function providerDecorator(target: undefined, context: ClassFieldDecoratorContext) {
    const propertyKey = context.name;

    context.addInitializer(function(this: BaseScriptComponent) {
      let component = null;
      for (let node = this.sceneObject; node !== null; node = node.getParent()) {
        component = node.getComponent(componentType);
        if (component !== null) break;
      }
      (this as any)[propertyKey] = component;
    });
  };
}
```
