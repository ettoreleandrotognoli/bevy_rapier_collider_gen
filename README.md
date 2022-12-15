# bevy_rapier_collider_gen

a library for generating bevy_rapier2d colliders, for bevy apps, from images with transparency

## example

![](/img/example-default.png)

to see this in action you can run the example, with no args it generates a scene with various colliders using pngs in the `assets/sprite` directory
```sh
cargo run --example colliders
```

you can also specify a path to an image yourself the example will attempt to generate one or more bevy_rapier_2d convex polylines for the objects it finds

## about / why

i was looking for a way to iterate on some 2d scenes with colliders on things with more sophisticated shapes than simple 
geometry, i figured there should be enough info in an image with transparency to generate colliders, and... there is! so i 
packaged up my approach here in case anyone else could benefit. 

## how it works

i was inspired by [a coding train (or, coding in the cabana rather) on an implementation of "marching squares"](https://youtu.be/0ZONMNUKTfU).
so this crate takes a "march through all the values" approach to find edges, i.e. pixels with at least 1 empty neighboring pixel, but 
instead of drawing a contour in place, it just keeps track of all the actual pixel coordinates. to determine "empty" I bitwise
or all the bytes for each pixel and, in images with transparency, "empty" is a zero value for the pixel.

after that, we need to put the coordinates in some kind of "drawing order" so whatever we pass all the points to, knows how we want the object constructed. for this, the
crate collects all pixels, in order, that are a distance of 1 from eachother. if there are pixels that have a distance greater than 1
from any pixel in an existing group, that pixel begins a new group.

## caveats

- as mentioned here and there in these docs, this implementation requires images to have transparency in order to distinguish object from non-object :)
- there's no reason we couldn't generate colliders / geometry without transparency, it's just not implemented. if you've got a compelling case, raise an issue! or even better, create a pr!
- i imagine for generating things at a larger scale, i.e. colliders for sets of sprites bigger than pixel counts in the hundreds, this implementation won't be performant to do at runtime. i'll suggest serializing the colliders you like and deserializing in your app instead of doing all the number crunching on load when you need a performance boost


## Examples of Colliders generated for assets/sprite/car.png 

(as in pictures of the sort of thing you can expect, not the runnable bevy app example. that's a couple headings up)


### convex polyline

![](/img/convex-polyline.png)

### polyline

![](/img/polyline.png)

### convex hull

![](/img/convex-hull.png)

### heightfield

the current implementation does best if the image you're generating a heightfield from is either centered in the image
or spans the entire width of the image...

![](/img/heightfield.png)

### convex decomposition

I didn't add support for convex decomposition directly because when sprites were small, and collisions were forceful, they were sort of unreliable (occasional panics because of bounds indexing in rapier's dependencies 💀). But if you wanted to use
convex decomposition colliders you could construct them with the edge coordinates from your image with something like this

```rust
let sprite_image = image_assets.get(sprite_handle.unwrap()).unwrap();
let edge_coordinate_groups = multi_image_edge_translated(sprite_image);
for coords in edge_coordinate_groups {
    let indices: Vec<[u32; 2]> = (0..coords.len()).map(|i| [i as u32, i as u32]).collect();
    let collider = Collider::convex_decomposition(&coords, &indices);
    commands.spawn((
        collider,
        RigidBody::Fixed,
        SpriteBundle {
            texture: sprite_handle.unwrap().clone(),
            ..default()
        },
    ));
}
```

![](/img/convex-decomposition.png)