# Eloquent sortable relations

![run-tests](https://github.com/re-thinking/eloquent-sortable-relations/workflows/run-tests/badge.svg?branch=master)
[![Software License](https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square)](LICENSE)
[![StyleCI](https://styleci.io/repos/261771207/shield?branch=master)](https://styleci.io/repos/261771207)

This package provides a new relation to Eloquent model

The value of the related model position column is determined by the maximum value of this column for parent record (through foreign key).

The package provides a relation class itself, traits for owner side and related side and an interface.

## Installation

This package can be installed through composer

```bash
composer require rethinking/eloquent-sortable-relations 
```

## Usage

To add sortable behaviour for your relations follow next steps:
1. Use `Rethinking\Eloquent\Relations\Sortable\HasSortedRelations` trait on the **owning** Model
2. Implement `Rethinking\Eloquent\Relations\Sortable\Sortable` interface on **related** Model. For simplicity, 
use `Rethinking\Eloquent\Relations\Sortable\HasSortingContext` trait
3. Define the *sorting context* for Model (the owning side)
4. Optionally define the column name for position attribute on your model (`position` by default in trait)
by overriding trait method `HasSortingContext::getPositionColumnName(): string`

### Example

Owning side:
```php
use Illuminate\Database\Eloquent\Model;
use Rethinking\Eloquent\Relations\Sortable\HasSortedRelations;

class Owner extends Model
{
    use HasSortedRelations;

    public function items()
    {
        return $this->hasManySorted(RelatedItem::class);
    }
}
```

Related side:
```php
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Rethinking\Eloquent\Relations\Sortable\HasSortingContext;
use Rethinking\Eloquent\Relations\Sortable\Sortable;

class RelatedItem extends Model implements Sortable
{
    use HasSortingContext;

    public function owner(): BelongsTo
    {
        return $this->belongsTo(Owner::class);
    }

    public function getSortingContext(): BelongsTo
    {
        return $this->owner();
    }
}
```

By default the sorting column name assumed to be _position_
For modifying this, override (in case trait is used) or implement the interface method
```php
public static function getPositionColumnName(): string
{
    return 'position';
}
```
For manually setting sorting order for existing items, use method `HasMnaySorted::setSortingOrder(iterable $ids, $start = 1)`
This will set the array index of $id + $start as position field
```php
//assume $owner has 3 related items with id 1,2,3 and positions of 1,2,3
$owner = Owner::find(1);

$owner->items()->setSortingOrder([2,3,1]);

$ids = $owner->items->pluck('id')->toArray(); //ids = [2,3,1]
``` 

In case, position sequence corrupted, you can manually trigger the `HasManySorted::resort()` method.
```php
//assume $owner has 3 related items with corresponding positions of [1,3,5]
$owner = Owner::find(1);

$owner->items()->resort();

$positions = $owner->items->pluck('position')->toArray(); //$positions = [1,2,3]
```

When creating the new relation for owning side, the position will be automatically calculated as maximum position value for owner foreign key
`max(position) + 1`.

Assume, owner has no related items. 
The calculation of the next position will be triggered when using any of the following methods:

```php
$owner = Owner::find(1);

$owner->items()->create();         //newly created item will have position = 1

$owner->items()->save(new Item()); //newly created item will have position = 2

$item = new Item();
$item->owner()->associate($owner);
$item->save();                     //newly created item will have position = 3
```

When dissociate the related model from it owner, the position will be reset to `null`
```php
$item = Item::find(2);

//assume $item has position = 2
$item->owner()->dissociate();

assert($item->position === null); //true
```

### Relations

Extension provides 3 types of sorted relations:

#### `HasManySorted`

Items will be fetched based on position within `Owner::class`:
```php
//assume $owner has 3 related items with id 1,2,3 and positions of 3,1,2
$owner = Owner::find(1);

$items = $owner->items()->get(); // items will be sorted by position i.e. ids will be [$item2,$item1,$item3]
```

#### `HasManySortedThrough`
Items will be fetched through `Middle::class` based on position of `Item::class` within `Middle::class`:
```php
//assume you have $owner -> $middle -> $item, where $item is sorted by $middle context

use Illuminate\Database\Eloquent\Model;
use Rethinking\Eloquent\Relations\Sortable\HasSortedRelations;
use Rethinking\Eloquent\Relations\Sortable\HasSortingContext;use Rethinking\Eloquent\Relations\Sortable\Sortable;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Owner extends Model
{
    use HasSortedRelations;

    public function middles()
    {
        return $this->hasMany(Middle::class);
    }
       
    public function items()
    {
        return $this->hasManySortedThrough(Item::class, Middle::class);
    }
}

class Middle extends Model 
{
    use HasSortedRelations;
    
    public function items()
    {
        return $this->hasManySorted(Item::class);
    }
    
    public function owner()
    {
        return $this->belongsTo(Owner::class);
    }
}

class Item extends Model implements Sortable
{
    use HasSortingContext;

    public function middle()
    {
        return $this->belongsTo(Middle::class);
    }
    
    public function getSortingContext(): BelongsTo
    {
        return $this->middle();
    }
}

//assume $owner has two $middle, each $middle has 3 $item
//items inside every $middle has sorting of [1,2,3]
$owner = Owner::find(1);

//items will be mixed i.e. $middle1Item1, $middle2Item2, $middle1Item2, $middle2Item2 etc.
$items = $owner->items()->get();
```

#### `HasManyThroughSorted`
Items will be fetched through `Middle::class` based on position of `Middle::class` within `Owner::class`:
```php
//assume you have $owner -> $middle -> $item, where $middle is sorted by $owner context

use Illuminate\Database\Eloquent\Model;
use Rethinking\Eloquent\Relations\Sortable\HasSortedRelations;
use Rethinking\Eloquent\Relations\Sortable\HasSortingContext;use Rethinking\Eloquent\Relations\Sortable\Sortable;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Owner extends Model
{
    use HasSortedRelations;

    public function middles()
    {
        return $this->hasMany(Middle::class);
    }
       
    public function items()
    {
        return $this->hasManySortedThrough(Item::class, Middle::class);
    }
}

class Middle extends Model implements Sortable
{
    use HasSortingContext;
    
    public function items()
    {
        return $this->hasMany(Item::class);
    }

    public function owner()
    {
        return $this->belongsTo(Owner::class);
    }

    public function getSortingContext() : BelongsTo
    {
        return $this->owner();
    }
}

class Item extends Model
{
    public function middle()
    {
        return $this->belongsTo(Middle::class);
    }
}

//assume $owner has two $middle, each $middle has 3 $item
//$middle1 has position 2
//$middle2 has position 1
$owner = Owner::find(1);

//items will be fetched from in order of $middle2 (position 1) and $middle1 (position 2)
$items = $owner->items()->get();
```

## Test

The package contains integration tests, set up with Orchestra. Tests can be run through composer script
```bash
composer test
```

## License

The MIT License (MIT). Please see [License File](LICENSE) for more information.
