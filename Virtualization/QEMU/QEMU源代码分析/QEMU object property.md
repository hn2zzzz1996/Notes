# QEMU object property

Properties are the external interface to an object.

```C
// include/qom/object.h
88 struct ObjectProperty
    89 {
    90     char *name;
    91     char *type;
    92     char *description;
    93     ObjectPropertyAccessor *get;
    94     ObjectPropertyAccessor *set;
    95     ObjectPropertyResolve *resolve;
    96     ObjectPropertyRelease *release;
    97     ObjectPropertyInit *init;
    98     void *opaque;
    99     QObject *defval;
   100 };
```

All properties are accessed through visitors:

```C
typedef void (ObjectPropertyAccessor)(Object *obj,
 Visitor *v, void *opaque,
 const char *name, Error **errp);
typedef void (ObjectPropertyRelease)(Object *obj,
 const char *name, void *opaque);
```

Similar to Linux sysfs, visitors instead of files.

## The object property APIs

```C
/*
 * Child properties form the composition tree.All objects need to be a child
 * of another object. Objects can only be a child of one object.
 */
ObjectProperty *
object_property_add_child(Object *obj, const char *name,
                          Object *child);
/**
 * object_property_add_str:
 * @obj: the object to add a property to
 * @name: the name of the property
 * @get: the getter or NULL if the property is write-only.  This function must
 *   return a string to be freed by g_free().
 * @set: the setter or NULL if the property is read-only
 *
 * Add a string property using getters/setters.  This function will add a
 * property of type 'string'.
 *
 * Returns: The newly added property on success, or %NULL on failure.
 */
ObjectProperty *object_property_add_str(Object *obj, const char *name,
                             char *(*get)(Object *, Error **),
                             void (*set)(Object *, const char *, Error **));

/**
 * object_property_add_bool:
 * @obj: the object to add a property to
 * @name: the name of the property
 * @get: the getter or NULL if the property is write-only.
 * @set: the setter or NULL if the property is read-only
 *
 * Add a bool property using getters/setters.  This function will add a
 * property of type 'bool'.
 *
 * Returns: The newly added property on success, or %NULL on failure.
 */
ObjectProperty *object_property_add_bool(Object *obj, const char *name,
                              bool (*get)(Object *, Error **),
                              void (*set)(Object *, bool, Error **));

/**
 * object_property_add_enum:
 * @obj: the object to add a property to
 * @name: the name of the property
 * @typename: the name of the enum data type
 * @lookup: enum value namelookup table
 * @get: the getter or %NULL if the property is write-only.
 * @set: the setter or %NULL if the property is read-only
 *
 * Add an enum property using getters/setters.  This function will add a
 * property of type '@typename'.
 *
 * Returns: The newly added property on success, or %NULL on failure.
 */
ObjectProperty *object_property_add_enum(Object *obj, const char *name,
                              const char *typename,
                              const QEnumLookup *lookup,
                              int (*get)(Object *, Error **),
                              void (*set)(Object *, int, Error **));

/**
 * object_property_add_uint8_ptr:
 * @obj: the object to add a property to
 * @name: the name of the property
 * @v: pointer to value
 * @flags: bitwise-or'd ObjectPropertyFlags
 *
 * Add an integer property in memory.  This function will add a
 * property of type 'uint8'.
 *
 * Returns: The newly added property on success, or %NULL on failure.
 */
ObjectProperty *object_property_add_uint8_ptr(Object *obj, const char *name,
                                              const uint8_t *v,
                                              ObjectPropertyFlags flags);

/**
 * object_property_add_uint16_ptr:
 * @obj: the object to add a property to
 * @name: the name of the property
 * @v: pointer to value
 * @flags: bitwise-or'd ObjectPropertyFlags
 *
 * Add an integer property in memory.  This function will add a
 * property of type 'uint16'.
 *
 * Returns: The newly added property on success, or %NULL on failure.
 */
ObjectProperty *object_property_add_uint16_ptr(Object *obj, const char *name,
                                    const uint16_t *v,
                                    ObjectPropertyFlags flags);

/**
 * object_property_add_uint32_ptr:
 * @obj: the object to add a property to
 * @name: the name of the property
 * @v: pointer to value
 * @flags: bitwise-or'd ObjectPropertyFlags
 *
 * Add an integer property in memory.  This function will add a
 * property of type 'uint32'.
 *
 * Returns: The newly added property on success, or %NULL on failure.
 */
ObjectProperty *object_property_add_uint32_ptr(Object *obj, const char *name,
                                    const uint32_t *v,
                                    ObjectPropertyFlags flags);

/**
 * object_property_add_uint64_ptr:
 * @obj: the object to add a property to
 * @name: the name of the property
 * @v: pointer to value
 * @flags: bitwise-or'd ObjectPropertyFlags
 *
 * Add an integer property in memory.  This function will add a
 * property of type 'uint64'.
 *
 * Returns: The newly added property on success, or %NULL on failure.
 */
ObjectProperty *object_property_add_uint64_ptr(Object *obj, const char *name,
                                    const uint64_t *v,
                                    ObjectPropertyFlags flags);

/**
 * object_property_add_alias:
 * @obj: the object to add a property to
 * @name: the name of the property
 * @target_obj: the object to forward property access to
 * @target_name: the name of the property on the forwarded object
 *
 * Add an alias for a property on an object.  This function will add a property
 * of the same type as the forwarded property.
 *
 * The caller must ensure that @target_obj stays alive as long as
 * this property exists.  In the case of a child object or an alias on the same
 * object this will be the case.  For aliases to other objects the caller is
 * responsible for taking a reference.
 *
 * Returns: The newly added property on success, or %NULL on failure.
 */
ObjectProperty *object_property_add_alias(Object *obj, const char *name,
                               Object *target_obj, const char *target_name);
```