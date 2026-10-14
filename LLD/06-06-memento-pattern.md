# Memento Pattern: Production Architecture & Low-Level Design in Java

> **Category:** [Behavioral Design Patterns](./06-behavioral-design-patterns.md) | **Navigation:** [Back to Behavioral Index](./06-behavioral-design-patterns.md) | [Master LLD Index](./00-index.md)

---


### 6.1 Intent, Architecture & Mechanics

The **Memento Pattern** captures and externalizes an object's internal state so that the object can be restored to this state later, **without violating encapsulation**. Private fields remain private; external caretakers cannot inspect or manipulate snapshot state.

Key roles:
1. **Originator**: The business object whose state must be captured and restored (`DocumentEditor`).
2. **Memento**: The immutable state snapshot (`DocumentSnapshot`). Only the Originator has access to its inner fields.
3. **Caretaker**: The external entity that stores historical mementos (`EditorHistoryManager`) without inspecting their contents.

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant Caretaker as EditorHistoryManager (Caretaker)
    participant Originator as DocumentEditor (Originator)
    participant Memento as DocumentSnapshot (Memento)

    User->>Originator: type("Hello World")
    User->>Caretaker: saveState()
    Caretaker->>Originator: createSnapshot()
    create participant Memento
    Originator->>Memento: new(title, content, cursor)
    Originator-->>Caretaker: snapshotInstance
    Caretaker->>Caretaker: pushToUndoStack(snapshotInstance)

    User->>Originator: type(" Corrupted Text")
    User->>Caretaker: undo()
    Caretaker->>Caretaker: popFromUndoStack()
    Caretaker->>Originator: restore(snapshotInstance)
    Originator->>Memento: readState()
    Originator->>Originator: overwriteState()
```

### 6.2 Production-Grade Java Implementation: Rich Text Editor with Memory-Optimized Snapshots

```java
package com.systemdesign.lld.behavioral.memento;

import java.time.Instant;
import java.util.Objects;

/**
 * The Memento: Strictly immutable snapshot. Package-private state access.
 */
public final class DocumentSnapshot {
    private final String title;
    private final String content;
    private final int cursorPosition;
    private final long timestampEpochMs;

    DocumentSnapshot(String title, String content, int cursorPosition) {
        this.title = title;
        this.content = content;
        this.cursorPosition = cursorPosition;
        this.timestampEpochMs = Instant.now().toEpochMilli();
    }

    String getTitle() { return title; }
    String getContent() { return content; }
    int getCursorPosition() { return cursorPosition; }
    public long getTimestampEpochMs() { return timestampEpochMs; }
}
```

```java
package com.systemdesign.lld.behavioral.memento;

import java.util.Objects;
import java.util.concurrent.locks.ReentrantReadWriteLock;

/**
 * The Originator: Maintains dynamic mutable state and creates/restores snapshots.
 */
public final class DocumentEditor {
    private String title;
    private StringBuilder content;
    private int cursorPosition;
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();

    public DocumentEditor(String initialTitle) {
        this.title = Objects.requireNonNull(initialTitle);
        this.content = new StringBuilder();
        this.cursorPosition = 0;
    }

    public void write(String text) {
        rwLock.writeLock().lock();
        try {
            content.insert(cursorPosition, text);
            cursorPosition += text.length();
        } finally {
            rwLock.writeLock().unlock();
        }
    }

    public void delete(int length) {
        rwLock.writeLock().lock();
        try {
            int start = Math.max(0, cursorPosition - length);
            content.delete(start, cursorPosition);
            cursorPosition = start;
        } finally {
            rwLock.writeLock().unlock();
        }
    }

    public void setTitle(String newTitle) {
        rwLock.writeLock().lock();
        try {
            this.title = Objects.requireNonNull(newTitle);
        } finally {
            rwLock.writeLock().unlock();
        }
    }

    public DocumentSnapshot createSnapshot() {
        rwLock.readLock().lock();
        try {
            return new DocumentSnapshot(title, content.toString(), cursorPosition);
        } finally {
            rwLock.readLock().unlock();
        }
    }

    public void restore(DocumentSnapshot snapshot) {
        Objects.requireNonNull(snapshot, "Cannot restore null snapshot");
        rwLock.writeLock().lock();
        try {
            this.title = snapshot.getTitle();
            this.content = new StringBuilder(snapshot.getContent());
            this.cursorPosition = snapshot.getCursorPosition();
        } finally {
            rwLock.writeLock().unlock();
        }
    }

    public String printDocument() {
        rwLock.readLock().lock();
        try {
            return String.format("[Title: %s | Cursor: %d]%nContent:%n%s", title, cursorPosition, content);
        } finally {
            rwLock.readLock().unlock();
        }
    }
}
```

```java
package com.systemdesign.lld.behavioral.memento;

import java.util.ArrayDeque;
import java.util.Deque;
import java.util.Objects;

/**
 * The Caretaker: Bounded history manager managing undo/redo snapshots.
 */
public final class EditorHistoryManager {
    private final DocumentEditor editor;
    private final Deque<DocumentSnapshot> undoStack = new ArrayDeque<>();
    private final Deque<DocumentSnapshot> redoStack = new ArrayDeque<>();
    private final int maxCapacity;

    public EditorHistoryManager(DocumentEditor editor, int maxCapacity) {
        this.editor = Objects.requireNonNull(editor);
        this.maxCapacity = maxCapacity;
    }

    public void checkpoint() {
        if (undoStack.size() >= maxCapacity) {
            undoStack.removeLast();
        }
        undoStack.push(editor.createSnapshot());
        redoStack.clear();
    }

    public boolean undo() {
        if (undoStack.isEmpty()) {
            return false;
        }
        redoStack.push(editor.createSnapshot());
        DocumentSnapshot snapshot = undoStack.pop();
        editor.restore(snapshot);
        return true;
    }

    public boolean redo() {
        if (redoStack.isEmpty()) {
            return false;
        }
        undoStack.push(editor.createSnapshot());
        DocumentSnapshot snapshot = redoStack.pop();
        editor.restore(snapshot);
        return true;
    }
}
```

```java
package com.systemdesign.lld.behavioral.memento;

public class MementoPatternDemo {
    public static void main(String[] args) {
        DocumentEditor editor = new DocumentEditor("System Architecture Specification");
        EditorHistoryManager history = new EditorHistoryManager(editor, 10);

        System.out.println("=== 1. Initial Draft and Checkpoint ===");
        editor.write("Section 1: Distributed Cache Architecture.\n");
        history.checkpoint();
        System.out.println(editor.printDocument());

        System.out.println("\n=== 2. Second Revision and Checkpoint ===");
        editor.write("Section 2: Raft Consensus Replication.\n");
        history.checkpoint();
        System.out.println(editor.printDocument());

        System.out.println("\n=== 3. Erroneous Edit (Unsaved) ===");
        editor.write("ERASED ENTIRE DOCUMENT BY MISTAKE!");
        System.out.println(editor.printDocument());

        System.out.println("\n=== 4. Undo Recovery ===");
        history.undo();
        System.out.println(editor.printDocument());

        System.out.println("\n=== 5. Undo Back to Version 1 ===");
        history.undo();
        System.out.println(editor.printDocument());
    }
}
```

### 6.3 Concurrency & Memory Optimizations
- **Encapsulation Protection**: `DocumentSnapshot` getters use package-private visibility (`String getTitle()`). Classes outside the `com.systemdesign.lld.behavioral.memento` package cannot inspect or mutate snapshot fields, preventing leakage of internal state.
- **Copy-on-Write Memory Guard**: Capturing full string contents on every keystroke in large documents induces garbage collection thrashing. Production implementations use immutable rope data structures or store incremental diffs (reverse patch edits).
- **Thread Safety via ReadWriteLock**: Read locks guard snapshot creation (`createSnapshot()`), while write locks guard restoration (`restore()`), ensuring atomic state capture.

---

---

## Navigation & Related Patterns
* [Back to Behavioral Patterns Index](./06-behavioral-design-patterns.md)
* [Master LLD Index](./00-index.md)
* [Previous: 06-05-mediator-pattern.md](./06-05-mediator-pattern.md)
* [Next: 06-07-observer-pattern.md](./06-07-observer-pattern.md)
