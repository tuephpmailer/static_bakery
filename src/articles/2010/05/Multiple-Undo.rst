Multiple Undo
=============

by shantamg on May 12, 2010

This plugin adds undo/redo functionality to any app using a very
simple database structure. I am new to cake and this is my first
attempt at making a contribution. It is in a very early state and I
would love to see what the experienced bakers can show me about it.
This project consists of a Component, a Behavior, and 3 Models which
all work together to provide unlimited undo/redo capabilities for any
database changes, by keeping track of those changes in 3 separate undo
tables.

This project has now been packaged as a plugin and is available here:
`http://github.com/shantamg/multiple_undo`_

Set Up
~~~~~~

1. Download the source from the repo linked above, make sure the
parent folder is called "multiple_undo", and place it in the "plugins"
folder in your app.

2. Create the database tables from the sql file located in
/multiple_undo/config/schema/tables.sql

3. Add the following to the appController :

::

    public $components = array('MultipleUndo.Undoer');

4. Add the following to your appModel :

::

    var $actsAs = array('MultipleUndo.Undo');



Usage
~~~~~


Initializing the Undo Tables
````````````````````````````

To clean the slate and reset the undo tables (i.e. when the a user
logs in), you can call:

::

    $this->Undoer->wipe();



Writing Undos For Your Transactions
```````````````````````````````````

Now, in any controller action, in order to tell the behavior to start
recording an undo, call:

::

    $this->Undoer->start();

Then, perform whatever creates/updates/deletes you want.

Finally, call:

::

    $this->Undoer->stop($description = null);

where $description is a string describing the transaction that was
preformed (which can be undone).

So, any time you wrap some model saves and deletes with (before a
redirect!) or the undo will not be properly saved.

Note: You can not undo an updateAll() because, as mentioned in the
Cookbook, updateAll() bypasses the beforeSave().

Now, Undo and Redo!
```````````````````

To undo or redo, you could put something like this in any of your
views (or even in the layout so that it is available from everywhere):

::

    <?=$html->link('undo', array('plugin' => 'multiple_undo', 'controller' => 'undos', 'action' => 'undo'));?>	
    <?=$html->link('redo', array('plugin' => 'multiple_undo', 'controller' => 'undos', 'action' => 'redo'));?>

The undo and redo actions in the UndosController redirect to the
referring page, but you can play around with that if you like.

One thing that I did not mention is the implementation of the
"description" of each undo. This description is optional, and could be
kept in the session and displayed as the user clicks back through
undos...



How It Works

+ The Undoer Component allows any controller to mark a database
  transaction (1 or more creates/updates/deletes) as "undoable."
+ The Undo Behavior checks the data in beforeSave and records the
  state of the rows that will change.
+ The 3 related models, Undo , UndoTables , and UndoFields . are used
  to store the undo data.

The data in the 3 undo tables accomplish two things:


#. Identify the model, what action was performed on it (create,
   update, delete), and the names and contents of each field before the
   action took place.
#. Group multiple database changes into one transaction, or "undo",
   and give it a description.



A Note About the Table ids: '0' is Always Now
`````````````````````````````````````````````

The id in the Undo model serves as a chronology of undoable actions.
But it also does more. Instead of storing somewhere a placeholder for
the current undo, we nudge the ids up and down as we traverse the
table so that 0 always represents the current undo. In fact, the id of
an undo represents its distance from the current action .


+ 0 is the first available undo.
+ Positive ids are increasingly older undos.
+ Negative ids are redos.

As actions are being performed and undos are being recorded, each undo
is born with id 0, and all undos are nudged up when a new '0' is made.
In other words, they are being created in such a way that their ids
are directly proportional to their age (the higher the id, the older
the entry).

When we perform an undo (always at id 0), we nudge all of the ids down
( -1). In this way, negative ids represent possible redos. When
performing a redo (always at id -1), we then nudge all of the ids up
(+1).


+ [li] When performing undos, the ids are nudged down (-1) [li] When
  performing redos, the ids are nudged up (+1)



Thoughts
My main intention here, as a newbie to cake, is to develop proper
baking skills. I tried to write this in the way that I thought made
sense, but what I really could use is some feedback about where my
code is and how it is organized.

Also, this feature is just what I needed for my app, but I can see how
it's usefulness in other apps might be rare. It really is made for one
user making changes to very certain tables on their own. In that
sense, this code should be expanded in order to accomodate multiple
users (a foreign key to the User model). My hope is that someone finds
it interesting enough to chew on with me.

Thanks for checking it out!


.. _http://github.com/shantamg/multiple_undo: http://github.com/shantamg/multiple_undo
.. meta::
    :title: Multiple Undo
    :description: CakePHP Article related to ,Plugins
    :keywords: ,Plugins
    :copyright: Copyright 2010 shantamg
    :category: plugins

