===================
Working with Events
===================

Sponge provides a system to:

- Listen for events
- Fire events

Overview
========

Events are used to inform plugins of certain happenings. Many events can also be *cancelled* -- that is, the action that
the event refers to can be prevented from occurring. Cancellable events implement the ``Cancellable`` interface.

Sponge itself contains many events; however, plugins can create their own events which other plugins can listen to.

Event listeners are assigned a priority that determines the order in which the event listener is run in context of other
event listeners (such as those from other plugins). For example, an event listener with *EARLY* priority will return
before most other event listeners.

Events cannot be sent to a specific set of plugins. All plugins that listen to an event will be notified of the event.

The event bus or event manager is the class that keeps track of which plugins are listening to which event,
and is also responsible for distributing events to event listeners.

.. note::
  The event bus **supports supertypes**. For example, ``BreakBlockEvent`` extends ``ChangeBlockEvent``.
  Therefore, a plugin could listen to ``ChangeBlockEvent`` and still receive ``BreakBlockEvent``\ s. However,
  a plugin listening to just ``BreakBlockEvent`` would not be notified of other types of ``ChangeBlockEvent``.

Event Listeners
===============

In order to listen for an event, an event listener must be registered. This is done by making a method with any name,
defining the first (and only) parameter to be the desired event type, and then affixing ``@Listener`` to the method,
as illustrated below.

.. code-block:: java

    import org.spongepowered.api.event.Listener;

    @Listener
    public void onSomeEvent(SomeEvent event) {
        // Do something with the event
    }

In addition, the class containing these methods must be registered with the event manager:

.. tip::

    For event listeners on your main plugin class (annotated by ``@Plugin``), you do not need to register the object for
    events because Sponge will do it automatically.



Registering and Unregistering Event Listeners
=============================================

To register event listeners annotated by ``@Listener`` that are not in the main plugin class, you can use
``EventManager#registerListeners``, which accepts a reference to the plugin and an instance
of the class containing the event listeners.

**Example: Registering Event Listeners in Other Classes**

.. code-block:: java

    public class ExampleListener {

        @Listener
        public void onBreakBlock(BreakBlockEvent event) {
            ...
        }
    }

    game.getEventManager().registerListeners(this, new Examplelistener());



Dynamically Registering Event Listeners
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Some plugins (such as scripting plugins) may wish to dynamically register an event listener. In that case the event
listener is not a method annotated with ``@Listener``, but rather a class implementing the ``EventListener`` interface.
This event listener can then be registered by calling ``EventManager#registerListener``, which accepts a reference to the
plugin as the first argument, the ``Class`` of events handled as the second argument, and the listener itself as the
final argument. Optionally, you can specify an ``Order`` to run the event listener in as the third argument or a
boolean value as the fourth argument (before the instance of the listener) which determines whether to call the listener
before other server modifications.

**Example: Implementing EventListener**

.. code-block:: java

    public class ExampleListener implements EventListener<BreakBlockEvent> {

        @Override
        public void handle(BreakBlockEvent event) throws Exception {
            ...
        }
    }

**Example: Dynamically Registering the Event Listener**

.. code-block:: java

    EventListener<BreakBlockEvent> listener = new ExampleListener();
    game.getEventManager().registerListener(this, BreakBlockEvent.class, listener);

.. tip::

        For event listeners created with the ``@Listener`` annotation, the order of the execution can be configured
        (see also `About @Listener`_). For dynamically registered listeners this is possible by passing an ``Order``
        to the third argument the ``EventManager#registerListener`` method.


Unregistering Event Listeners
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To unregister a single event listener, you can use the ``EventManager#unregisterListeners`` method, which accepts
an instance of the class containing the event listeners.

.. code-block:: java

    EventListener listener = ...
    game.getEventManager().unregisterListeners(listener);

Alternatively, you can use ``EventManager#unregisterPluginListeners``, passing in a reference to the plugin, to
unregister all event listeners associated with that plugin. Note that this will remove *all* of the plugin's event
listeners, including those registered with ``@Listener`` annotations.

.. code-block:: java

    MyPlugin plugin = ...
    game.getEventManager.unregisterPluginListeners(plugin);

About @Listener
~~~~~~~~~~~~~~~~

The ``@Listener`` annotation has a few configurable fields:

* ``order`` is the order in which the event listener is to be run. See the ``org.spongepowered.api.event.Order`` enum
  in Sponge to see the available options.
* ``ignoreCancelled``, if true (which is default true), causes the event listener to be skipped if the event in question
  is cancellable and has been cancelled (by a previously-executed plugin, for example).

By default, ``@Listener`` is configured so that your event listener will *not* be called if the event in question is
cancellable and has been cancelled (such as by another plugin).

Firing Events
=============

To dispatch an event, you need an object that implements the ``org.spongepowered.api.event.Event`` interface.

You can fire events using the event bus (``org.spongepowered.api.service.event.EventManager``):

.. code-block:: java

    boolean cancelled = game.getEventManager().post(theEventObject);

The method returns ``true`` if the event was cancelled, ``false`` if not.

Firing Sponge Events
~~~~~~~~~~~~~~~~~~~~

It is possible to generate instances of built-in events with the static ``SpongeEventFactory``.


Creating Custom Events
======================

You can write your own event classes and dispatch those events using the method described above.

An event class must either implement the ``Event`` interface or extend the ``AbstractEvent`` class.

If you want your event to be cancellable, the class must also implement ``Cancellable``.

Example: Custom Event Class
~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: java

    package example.event;

    import org.spongepowered.api.entity.player.Player;
    import org.spongepowered.api.event.impl.AbstractEvent;
    import org.spongepowered.api.event.Cancellable;

    public class PrivateMessageEvent extends AbstractEvent implements Cancellable {

       private boolean cancelled = false;

       private Player sender;
       private Player recipient;

       private String message;

       public Player getSender() {
          return sender;
       }

       public Player getRecipient() {
          return recipient;
       }

       public String getMessage() {
          return message;
       }

       @Override
       public boolean isCancelled() {
          return cancelled;
       }

       @Override
       public void setCancelled(boolean cancel) {
          cancelled = cancel;
       }

       public PrivateMessageEvent(Player sender, Player recipient, String message) {
          this.sender = sender;
          this.recipient = recipient;
          this.message = message;
       }
    }


Example: Fire Custom Event
~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: java

    game.getEventManager().post(new PrivateMessageEvent(playerA, playerB, "Hello World!"));


Example: Listen for Custom Event
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: java

    @Listener
    public void onPrivateMessage(PrivateMessageEvent event) {
        if(event.getMessage().equals("hi i am from planetminecraft")) {
           event.setCancelled();
           return;
        }

        String senderName = event.getSender().getName();
        event.getReceipient().sendMessage(ChatTypes.CHAT, "PM from " + senderName + ": " + event.getMessage());
    }

Callbacks
=========

Callbacks are a more advanced feature of Sponge's event system.

Callbacks allow plugins to cooperate better when they override vanilla behavior. When an event is invoked, Sponge runs
through the event listeners in order from first to last. Then Sponge runs through the callback list in order from last
to first. Vanilla is always the first callback added, meaning that vanilla's listener will be executed last.

Plugins that don't use callbacks can also use the simpler ``setCancelled(boolean)`` method, which will disable all
callbacks. However, some plugins may just need to disable vanilla behavior, modify another plugin's behavior, or disable
that behavior completely. These are cases where the flexibility offered through callbacks is required.

A plugin can add as many callbacks as it needs during an event, and plugins can cancel specific callbacks. However, a
plugin cannot reorder or remove callbacks, as some behaviors (especially vanilla) cannot be reordered. Additionally, all
modifications to the callback list, should be done in the event listener itself. Attempting to change the list during
callback execution will cause a ``ConcurrentModificationException``. Callbacks should only be added or cancelled in
event listeners who's ``Order`` property allows event cancellation.

.. note::

    ``ExplosionEvent`` doesn't exist in the API currently, it is just used for example purposes.


Example: Adding a Callback to Disable Explosions and Spawn an Arrow
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. note::

    This is a bad example, but use-cases for callbacks are going to be very specific so this just demonstrates the code
    needed to add one.

.. code-block:: java

    @Listener
    // final not required unless using an inner class that needs access to it.
    public void onExplosion(final ExplosionEvent event) {
        for (EventCallback callback : event.getCallbacks()) {
            // Disable vanilla behavior
            if (callback.isBaseGame()) {
                if (callback instanceof Cancellable) {
                    ((Cancellable) callback).setCancelled(true);
                }
            }
        }

        event.getCallbacks().add(new EventCallback() {
            public boolean isBaseGame() {
                // Not a base game (i.e. Vanilla) behavior
                return false;
            }

            public void run() {
                Extent extent = event.getEntity().getLocation().getExtent();

                // Create an arrow
                extent.createEntity(EntityTypes.ARROW, event.getEntity().getLocation().getPosition());
            }
        });
    }

Example: Disable Chair Sitting Added by CraftBook
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. note::

    This example will break if other plugins enable or disable callbacks.

.. code-block:: java

    @Listener
    public void onPlayerInteractBlock(PlayerInteractBlockEvent event) {
        boolean foundChair = false;

        for (Callback callback : event.getCallbacks())
            if (callback instanceof com.sk89q.craftbook.mechanic.Chair) {
                if (callback instanceof Cancellable) {
                    ((Cancellable) callback).setCancelled(true);
                }
                foundChair = true;
                break;
            }
        }

        if (foundChair) {
            for (Callback callback : event.getCallbacks()) {
                if (!(callback instanceof com.sk89q.craftbook.mechanic.Chair)) {
                  if (callback instanceof Cancellable) {
                      ((Cancellable) callback).setCancelled(false);
                  }
                }
            }
        }
    }

Example: Modifying Behaviors
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: java


    @Listener
    public void onExplosion(ExplosionEvent event) {
        for (Callback callback : event.getCallbacks()) {
            if (callback instanceof example.FireworksExplosion) {
                ((example.FireworksExplosion) callback).setYield(200);
            }
        }
    }



Thanks to @sk89q for the callback examples. They were copied from his original
`PR #232 <https://github.com/SpongePowered/SpongeAPI/pull/232>`_.
