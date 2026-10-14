# Prototype Pattern: Production Architecture & Low-Level Design in Java

> **Category:** [Creational Design Patterns](./04-creational-design-patterns.md) | **Navigation:** [Back to Creational Index](./04-creational-design-patterns.md) | [Master LLD Index](./00-index.md)

---


### 5.1 Intent, Problem Context & Architecture
The **Prototype Pattern** specifies the kinds of objects to create using a prototypical instance, and creates new objects by copying this prototype.

It is useful when direct object creation is computationally expensive (e.g., requires complex database lookups, disk I/O, heavy mathematical precomputations, or large configuration graph parsing).

```
+-------------------------------------------------------------+
|                          Prototype                          |
+-------------------------------------------------------------+
| + cloneDeep(): Prototype                                    |
+-------------------------------------------------------------+
                               ^
                               |
+-------------------------------------------------------------+
|                  RouterConfigurationTemplate                 |
+-------------------------------------------------------------+
| - deviceHostname: String                                    |
| - routingTable: List<RouteEntry>                            |
| - firewallRules: Map<String, Rule>                          |
+-------------------------------------------------------------+
| + cloneDeep(): RouterConfigurationTemplate                  |
+-------------------------------------------------------------+
```

---

### 5.2 Shallow Copy vs Deep Copy in Java
* **Shallow Copy**: Creates a new object, but copies field values as-is. Object references inside point to the exact same memory locations as the original object. Mutating an inner object in the clone mutates the original object.
* **Deep Copy**: Recursively duplicates the entire object graph. The clone has completely independent heap allocations for all referenced child objects.

```
Shallow Copy:
Original [ Config ] -----> [ RoutingTable List (Address: 0x889) ]
Clone    [ Config ] -----------^

Deep Copy:
Original [ Config ] -----> [ RoutingTable List (Address: 0x889) ]
Clone    [ Config ] -----> [ RoutingTable List (Address: 0x992) ] (Fresh Heap Memory)
```

#### Why Java's Built-in `Cloneable` and `Object.clone()` are Flawed
1. `Cloneable` is a marker interface that contains no methods; `clone()` is protected on `java.lang.Object`.
2. It bypasses constructors completely, creating objects in abnormal ways.
3. It defaults to a shallow copy.
4. Handling `final` fields during deep cloning is nearly impossible using `clone()` because `final` fields cannot be reassigned outside constructors.
5. It throws the checked exception `CloneNotSupportedException`.
6. **Best Practice**: Use **Copy Constructors** or an explicit, strongly-typed `copy()` / `deepClone()` method.

---

### 5.3 Production Implementation: Network Configuration Template Registry

```java
package com.systemdesign.creational.prototype;

import java.util.*;

public record RouteEntry(String destinationSubnet, String nextHopIp, int metric) {
    public RouteEntry {
        Objects.requireNonNull(destinationSubnet);
        Objects.requireNonNull(nextHopIp);
    }
}

public interface Prototype<T> {
    T deepClone();
}

public final class RouterConfiguration implements Prototype<RouterConfiguration> {

    private String deviceHostname;
    private final String firmwareVersion;
    private final List<RouteEntry> routingTable;
    private final Map<String, String> interfaces;

    public RouterConfiguration(String deviceHostname, String firmwareVersion, 
                               List<RouteEntry> routingTable, Map<String, String> interfaces) {
        this.deviceHostname = Objects.requireNonNull(deviceHostname);
        this.firmwareVersion = Objects.requireNonNull(firmwareVersion);
        this.routingTable = new ArrayList<>(Objects.requireNonNull(routingTable));
        this.interfaces = new HashMap<>(Objects.requireNonNull(interfaces));
    }

    // Copy Constructor for Deep Copying
    public RouterConfiguration(RouterConfiguration source) {
        this.deviceHostname = source.deviceHostname;
        this.firmwareVersion = source.firmwareVersion; // String is immutable
        
        // Deep copy collections
        this.routingTable = new ArrayList<>(source.routingTable.size());
        for (RouteEntry entry : source.routingTable) {
            // Records are immutable, so copying the record reference is thread-safe
            this.routingTable.add(new RouteEntry(entry.destinationSubnet(), entry.nextHopIp(), entry.metric()));
        }

        this.interfaces = new HashMap<>(source.interfaces);
    }

    @Override
    public RouterConfiguration deepClone() {
        return new RouterConfiguration(this);
    }

    public void setDeviceHostname(String deviceHostname) {
        this.deviceHostname = deviceHostname;
    }

    public void addRoute(RouteEntry entry) {
        this.routingTable.add(entry);
    }

    public void assignInterfaceIp(String iface, String ip) {
        this.interfaces.put(iface, ip);
    }

    public List<RouteEntry> getRoutingTable() {
        return Collections.unmodifiableList(routingTable);
    }

    public Map<String, String> getInterfaces() {
        return Collections.unmodifiableMap(interfaces);
    }

    public String getDeviceHostname() {
        return deviceHostname;
    }

    public String getFirmwareVersion() {
        return firmwareVersion;
    }
}

/**
 * Prototype Registry storing pre-warmed, pre-validated configuration templates.
 */
public final class NetworkTemplateRegistry {

    private final Map<String, RouterConfiguration> registry = new HashMap<>();

    public void registerTemplate(String templateKey, RouterConfiguration config) {
        registry.put(templateKey, config);
    }

    public RouterConfiguration createConfigFromTemplate(String templateKey, String newHostname) {
        RouterConfiguration template = registry.get(templateKey);
        if (template == null) {
            throw new IllegalArgumentException("No template registered with key: " + templateKey);
        }
        // Deep clone prototype and customize
        RouterConfiguration clone = template.deepClone();
        clone.setDeviceHostname(newHostname);
        return clone;
    }
}
```

---

---

## Navigation & Related Patterns
* [Back to Creational Patterns Index](./04-creational-design-patterns.md)
* [Master LLD Index](./00-index.md)
* [Previous: 04-04-builder-pattern.md](./04-04-builder-pattern.md)
* [Next: 04-06-object-pool-pattern.md](./04-06-object-pool-pattern.md)
