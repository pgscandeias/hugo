+++
date = "2016-04-04T15:09:13+02:00"
draft = true
title = "Simplify"

+++

Let's take a look at how easy it is to start out with the best of intentions, following [SOLID](https://en.wikipedia.org/wiki/SOLID_(object-oriented_design)) principles, and then quickly lose control. By adhering more closely to [Single Responsibility](https://en.wikipedia.org/wiki/Single_responsibility_principle) and enlisting the help of [Open/Closed](https://en.wikipedia.org/wiki/Open/closed_principle), we then solve the mess.


## It starts out innocently enough

So I have this `CtaService` class.

It needs to provide calls to action to a user, based on events that happened to said user. We know there are many different kinds of events, which leads to different kinds of Ctas, so we delegate creation responsibility to a set of factories.

```php
interface EventStore
{
    public function getByUser(User $user);
}

interface CtaFactory
{
    public function canHandle(Event $event);
    public function getCta(Event $event);
}

class CtaService
{
    protected $events;
    protected $factories = [];

    public function __construct(EventStore $events, array $factories)
    {
        $this->events = $events;
        $this->factories = $factories;
    }

    public function getCtas(User $user)
    {
        $ctas = [];
        foreach ($this->events->getByUser($user) as $event) {
            foreach ($this->factories as $factory) { // ugh
                if ($factory->canHandle($event)) {
                    $ctas[] = $factory->getCta($event);
                }
            }
        }

        return $ctas;
    }
}
```

Everyting is nice and clean and life is good.

## But then it gets weird

A project manager rolls in and he's like

> Most users are expected to get a list of `Cta`s no matter their events.
 
Fine. Let's call them base ctas and always retrieve them before the other ones.

```php
class CtaService
{
    public function getCtas(User $user)
    {
        $ctas = $this->getBaseCtas(User $user);
        ...
    }

    protected function getBaseCtas(User $user)
    {
        ...
    }
}
```

Nailed it!

Now, where do these base ctas come from? Let's say they depend on some `StatusService` which we now need to inject.

```php
class CtaService
{
    public function __construct(
        EventStore $events, 
        array $factories,
        StatusService $status
    ) {
        $this->events = $events;
        $this->factories = $factories;
        $this->status = $status;
    } {}

    # ...
    
    protected function getBaseCtas(User $user)
    {
        $ctas = [];
        if ($this->status->isVIP()) {
            $ctas[] = new VIPStatusCta();
        }

        return $cta;
    }

}
```

Sorted! But, you know, now the air is a little funky around this piece of code.


## And it gets weirder

Ok, so now it's 2 weeks later and I'm told we need to give the customer a call to action to buy the most expensive product they can afford:

Um, I guess maybe...

```php
class CtaService
{
    public function __construct(
        EventStore $events, 
        array $factories,
        StatusService $status,
        ProductStore $products
    ) {
        # ...
        $this->products = $products;
    } {}

    # ...
    
    protected function getBaseCtas(User $user)
    {
        $ctas = [];

        $product = $this
            ->products
            ->getMostExpensiveBelow($user->getBalance())
        ;
        if ($product) {
            $ctas[] = new ProductCta($product);
        }

        if ($this->status->isVIP()) {
            $ctas[] = new VIPStatusCta();
        }

        return $cta;
    }

}
```

And this is how my face ends up pasted on a dartboard.


## Thinking cap on

So what's going on here? How did this simple class get so out of hand so quickly? 

### Error #1 - we failed to design anything

Let's be honest, we never designed our `CtaService`, it just sort of happened. We needed to get Ctas as quickly as possible, so we coded something up that gave us Ctas.

### Error #2 - we asked too much from a single class

It was clear from the beginning that this service was always have to delegate actual `Cta` creation to some collaborators. We started out well, but then succumbed to the temptation of keeping it simple by solving each new requirement with its own ad-hoc piece of code, bloating the `CtaService` and its methods.


## A solution

Keeping a clear head is very important in software development. This could easily have been avoided by following the SOLID principles. Keeping responsibilities singular, ensuring classes are closed for modification but open to extension, and focusing on how objects communicate rather than how they transform data until we really have to.

**Q**: So, what should `CtaService` actually do?  
**A**: Delegate, delegate, delegate.

```php
interface BaseCtaFactory
{
    public function getCta(User $user);
}

class ProductCtaFactory implements BaseCtaFactory
{
    public function __construct(ProductStore $products) {}
    public function getCta(User $user)
    {
        $product = $this
            ->products
            ->getMostExpensiveBelow($user->getBalance())
        ;
        if ($product) {
            return new ProductCta($product);
        }
    }
}

interface EventCtaFactory
{
    public function canHandle(Event $event);
    public function getCta(Event $event);
}

class CtaService
{
    public function __construct(
        EventStore $events,
        array $baseFactories,   // BaseCtaFactory[]
        array $eventFactories,  // EventCtaFactory[]
    ) {}

    public function getAllCtas(User $user)
    {
        return array_merge(
            $this->getBaseCtas($user),
            $this->getEventCtas($user)
        );
    }

    public function getBaseCtas(User $user)
    {
        # Iterate $this->baseFactories, 
        # passing the user to each.
    }

    public function getEventCtas(User $user)
    {
        # Get events for the user, 
        # then iterate $this->baseFactories
        # and pass every event to every one 
        # (naive implementation).
    }
}
```

This service doesn't actually do anything of consequence anymore. It doesn't create or modify Ctas. It's pushing that responsibility down the line, which on the surface appears like just more complexity.

Thing is, though, now we're free to inject as many or as few `BaseCtaFactory` and `EventCtaFactory` instances as we need, meaning this class will probably never need to be changed again. Also, each factory is free to work in whatever way is needed. Each can be unit tested individually.

This code is, in essence, infinitely flexible and expansible. And did it really cost us that much development time? Did it really? I don't think so.

Software development gets very complicated very quickly not because we're dealing with hard problems (99% of all code is CRUD anyway, at least in web applications), but because we fail to put in the tiny little extra effort of making things maintainable from the start. Then we end up with huge balls of mud no one likes working with.

Following the SOLID principles really, _really_ helps.
