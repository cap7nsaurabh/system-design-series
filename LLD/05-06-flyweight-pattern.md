# Flyweight Pattern: Production Architecture & Low-Level Design in Java

> **Category:** [Structural Design Patterns](./05-structural-design-patterns.md) | **Navigation:** [Back to Structural Index](./05-structural-design-patterns.md) | [Master LLD Index](./00-index.md)

---


### 6.1 Intent & Memory Mechanics
The **Flyweight Pattern** uses sharing to support large numbers of fine-grained objects efficiently. In high-density applications (e.g., word processors, CAD software, particle rendering), creating millions of individual object instances exhausts JVM heap memory and triggers severe Garbage Collection pause times.

#### Intrinsic vs Extrinsic State
* **Intrinsic State**: Invariant, context-independent data stored permanently inside the Flyweight object. It is immutable and shareable across millions of contexts (e.g., character unicode, font family, style flags).
* **Extrinsic State**: Variant, contextual data passed into the Flyweight method by the client at execution time (e.g., screen X/Y coordinates, line number, selection state, render color).

```
[ Client Document Engine ]
       |
       | passes Extrinsic State (x=10, y=25, color=RED)
       v
[ GlyphFlyweight ('A', Arial, 12pt) ] <--- Shared among 100,000 occurrences in document!
```

---

### 6.2 Production Implementation: High-Performance Rich Text Glyph Engine

```java
package com.systemdesign.structural.flyweight;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

// 1. Intrinsic Flyweight: Immutable Glyph Formatter
public final class GlyphFlyweight {

    private final char character;
    private final String fontFamily;
    private final int fontSize;
    private final boolean isBold;
    private final boolean isItalic;

    public GlyphFlyweight(char character, String fontFamily, int fontSize, boolean isBold, boolean isItalic) {
        this.character = character;
        this.fontFamily = Objects.requireNonNull(fontFamily);
        this.fontSize = fontSize;
        this.isBold = isBold;
        this.isItalic = isItalic;
    }

    /**
     * Renders glyph using passed extrinsic parameters.
     */
    public void render(int x, int y, String hexColor, boolean isHighlighted) {
        System.out.printf("Render '%c' [Font: %s %dpt%s%s] at (%d,%d) Color: %s Highlight: %b%n",
                character, fontFamily, fontSize,
                (isBold ? " BOLD" : ""), (isItalic ? " ITALIC" : ""),
                x, y, hexColor, isHighlighted);
    }

    public char getCharacter() { return character; }
}

// 2. Thread-Safe Flyweight Factory
public final class GlyphFactory {

    private final Map<String, GlyphFlyweight> cache = new ConcurrentHashMap<>();

    public GlyphFlyweight getGlyph(char character, String fontFamily, int fontSize, boolean isBold, boolean isItalic) {
        String cacheKey = String.format("%c:%s:%d:%b:%b", character, fontFamily, fontSize, isBold, isItalic);

        return cache.computeIfAbsent(cacheKey, key -> {
            System.out.printf("[GlyphFactory] Creating new Intrinsic Flyweight for key: %s%n", key);
            return new GlyphFlyweight(character, fontFamily, fontSize, isBold, isItalic);
        });
    }

    public int getFlyweightPoolSize() {
        return cache.size();
    }
}

// 3. Extrinsic State Context
public record FormattedCharacter(GlyphFlyweight glyph, int x, int y, String color, boolean isSelected) {
    public void draw() {
        glyph.render(x, y, color, isSelected);
    }
}

// 4. Document Renderer Demonstrating Memory Savings
public final class RichTextDocument {

    private final GlyphFactory factory;
    private final List<FormattedCharacter> characters = new ArrayList<>();

    public RichTextDocument(GlyphFactory factory) {
        this.factory = Objects.requireNonNull(factory);
    }

    public void insertText(String text, String font, int size, boolean bold, boolean italic, int startX, int startY) {
        int currentX = startX;
        for (char c : text.toCharArray()) {
            GlyphFlyweight flyweight = factory.getGlyph(c, font, size, bold, italic);
            characters.add(new FormattedCharacter(flyweight, currentX, startY, "#000000", false));
            currentX += 8; // Advance cursor
        }
    }

    public void renderDocument() {
        for (FormattedCharacter fc : characters) {
            fc.draw();
        }
    }

    public int getTotalCharacterCount() { return characters.size(); }
    public int getUniqueFlyweightsAllocated() { return factory.getFlyweightPoolSize(); }
}
```

#### Memory Comparison Analysis
Suppose an e-book contains **1,000,000 characters**.
* **Without Flyweight**: 1,000,000 independent objects containing font names, size metrics, and metadata. At ~64 bytes per object, this consumes ~**64 MB** of heap memory just for character structures.
* **With Flyweight**: The 1,000,000 characters share approximately **100 distinct Flyweight instances** (uppercase, lowercase, digits, punctuation for 1-2 fonts). 100 objects consume ~**6.4 KB**.

---

### 6.3 Real-World Enterprise Usage
* **JDK `Integer.valueOf(int)`**: Pre-allocates and caches `Integer` instances in the range `[-128, 127]`.
* **JDK `String.intern()`**: Leverages the JVM String Pool to ensure duplicate string literals share a single canonical memory address.

---

---

## Navigation & Related Patterns
* [Back to Structural Patterns Index](./05-structural-design-patterns.md)
* [Master LLD Index](./00-index.md)
* [Previous: 05-05-facade-pattern.md](./05-05-facade-pattern.md)
* [Next: 05-07-proxy-pattern.md](./05-07-proxy-pattern.md)
