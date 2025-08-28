# Java Deep Object Sanitizer for Primitive char Fields Using VarHandle (Better Than Reflection)
Java Deep Object Sanitizer for Primitive char Fields Using VarHandle (High-Performance Alternative to Reflection)

If you’ve ever had to sanitize char fields in Java objects (for example, replacing invalid \u0000 values with a space ' '), you’ve probably run into two problems:

Primitive char fields can’t be null, but deserialization / mapping often leaves them as \u0000 (the default).

Reflection (Field#getChar / setChar) works but is slow and not JIT-friendly.

A better solution is to use Java VarHandles with caching. VarHandles give you near-direct access speed (much faster than reflection) and can be cached per class for reuse.

Full Example: Deep Char Sanitizer


package com.example.util;

import java.lang.invoke.MethodHandles;
import java.lang.invoke.VarHandle;
import java.lang.reflect.Field;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Deep object graph sanitizer for primitive char fields.
 * Replaces '\u0000' with ' '.
 * - Uses VarHandle (faster than reflection).
 * - Caches VarHandles per class for performance.
 * - Traverses nested objects, arrays, collections, and maps.
 */
public final class CharFieldSanitizer {

    private static final Map<Class<?>, List<VarHandle>> CACHE = new ConcurrentHashMap<>();
    private static final char REPLACEMENT = ' ';

    private CharFieldSanitizer() {}

    public static void sanitize(Object root) {
        if (root == null) return;
        sanitizeObject(root, new IdentityHashMap<>());
    }

    private static void sanitizeObject(Object obj, IdentityHashMap<Object, Boolean> seen) {
        if (obj == null || seen.put(obj, Boolean.TRUE) != null) return;

        Class<?> type = obj.getClass();

        if (isLeaf(type)) return;

        // sanitize primitive char fields
        for (VarHandle vh : handlesFor(type)) {
            char v = (char) vh.get(obj);
            if (v == '\u0000') vh.set(obj, REPLACEMENT);
        }

        // traverse nested
        for (Field f : type.getDeclaredFields()) {
            f.setAccessible(true);
            try {
                Object val = f.get(obj);
                if (val == null) continue;

                if (f.getType().isArray()) {
                    if (f.getType().getComponentType() == char.class) {
                        char[] arr = (char[]) val;
                        for (int i = 0; i < arr.length; i++) {
                            if (arr[i] == '\u0000') arr[i] = REPLACEMENT;
                        }
                    } else {
                        Object[] arr = (Object[]) val;
                        for (Object e : arr) sanitizeObject(e, seen);
                    }
                } else if (val instanceof Collection<?> col) {
                    for (Object e : col) sanitizeObject(e, seen);
                } else if (val instanceof Map<?, ?> map) {
                    for (Object e : map.values()) sanitizeObject(e, seen);
                } else if (!isLeaf(f.getType())) {
                    sanitizeObject(val, seen);
                }
            } catch (IllegalAccessException ignore) {}
        }
    }

    private static List<VarHandle> handlesFor(Class<?> type) {
        return CACHE.computeIfAbsent(type, cls -> {
            List<VarHandle> list = new ArrayList<>();
            try {
                MethodHandles.Lookup lookup = MethodHandles.privateLookupIn(cls, MethodHandles.lookup());
                for (Field f : cls.getDeclaredFields()) {
                    if (f.getType() == char.class) {
                        f.setAccessible(true);
                        list.add(lookup.unreflectVarHandle(f));
                    }
                }
            } catch (IllegalAccessException e) {
                // ignore
            }
            return List.copyOf(list);
        });
    }

    private static boolean isLeaf(Class<?> t) {
        return t.isPrimitive()
            || t == String.class
            || Number.class.isAssignableFrom(t)
            || t == Character.class
            || t == Boolean.class
            || t.isEnum()
            || t.getName().startsWith("java.time.");
    }
}


Why VarHandle is Better Than Reflection

Reflection (Field#getChar) is 10–100x slower per access.

VarHandle with caching is only ~2–5x slower than direct field access (and ~10–20x faster than reflection).

JIT can inline VarHandle calls but not reflection.

This approach is safe for high-throughput production if you call it at system boundaries (e.g. before serialization).
