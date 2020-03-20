# Lens Cheat Sheet

Accessors
=========

Access a field:

    thing ^. field

Go deeper:

    thing ^. field . field2

Transforming the result:

    thing ^. field . to (*2)

Fold the result (if it's foldable):

    thing ^. field . each

And do something to each value:

    thing ^. field . each . to (*2)


Don't fold (return the elements):

    thing ^.. field . each . to (*2)

Monadically (get a filename and return the handle):

    thing ^! field . act (\x -> openFile x ReadMode)

Monadically, without folding (get a list of filenames, and return a list of handles):

    thing ^!! field . each . act (\x -> openFile x ReadMode)


Defining
========

A field accessor's typeclass looks like this:

    class Has_field a b | a -> b where
        field :: Lens' a b

An instance is a combination of getter and setter, using a simple combinator:

    -- say we have Foo String String, and `field` refers to the first one
    instance Has_field Foo String where
        field = lens getter setter
            where getter (Foo f1 _)     = f1
                  setter (Foo f1 f2) v  = Foo v f2