# fast-multipole-method.js
This is a javascript implementation for the [fast multipole method](https://en.wikipedia.org/wiki/Fast_multipole_method). Classes exist for both 2D and 3D simulations. A subset of classes allow you to work easily with Vector2 and Vector3 objects in Three.js. Otherwise, you can work with any object that has x and y attributes, or alternatively, you can work with lists of size 2 or 3.

## The problem
Let's say you want to simulate a solar system. You'll recall all objects in the universe effect one another through gravity, right? A naïve simulator might simulate this as follows:

    for every object in the universe:
    	for every other object in the universe:
    		calculate the force exerted between the objects

The simulation above doesn't scale well as we increase the number of objects. If I want to simulate 3 objects I have to make 6 calculations per timestep. If I want to simulate 4 objects I have to make 12 calculations per timestep - I doubled the number of calculations just by adding one object! Things get out of hand quickly. What I really want is for the number of calculations to increase *in direct proportion* to the number of objects.

Well, fine then, why don't I do it this way:

    for every object in the universe:
    	for every location in the universe:
    		calculate the force exerted by the object from that location and store it in a grid somewhere
    
The "grid" we mention is an approximation for what physicists call a [field](https://en.wikipedia.org/wiki/Field_(physics)). For every point on a field, there is a corresponding value. Sometimes, that value is a scalar, like density or pressure. Sometimes, that value is a vector, like force, velocity, or acceleration. Gravity can be represented by a field of vectors, each vector expressing the acceleration one would experience at a specific location.

Now I'm a bit of a smart-aleck, here. It's true the solution above scales well with the number of objects, but it doesn't scale well with the size of the grid. If I want to simulate a square universe that's 3 grid cells wide, I have to make 9 calculations for each object. If I want to simulate a universe that's 4 grid cells wide I have to make 16 calculations for each object. Again, things get out of hand quickly. What we really need is a simulation that scales well for both the number of objects *and* the size of the grid.

Well, fine then, how about this. Gravity gets weaker with distance, so let's say we keep the grid, but we ignore any interaction between objects that are further than a grid cell's width apart. This approach scales well, but it has some undesireable consequences. Gravity might get weaker with distance in the real world, but it never completely vanishes. If we have a really massive object, that object will no longer be able to effect distant objects like it should.

## The fast multipole method
The solution here relies upon an observation. As you move further away from an object, you still have to simulate gravity, but it becomes less important to do so accurately. This is because gravity follows an [inverse square law](https://en.wikipedia.org/wiki/Inverse-square_law). The further you go, the less gravity you feel, and the less important it becomes to get the details right. The only thing that continues to matter is the order of magnitude. 

An object that's very far away is best modeled using very large grid cells. Only a single value is stored per grid cell, so it's not likely this value will correctly represent the force of gravity all throughout its cell, but that's okay, because the object is far away and accuracy doesn't matter much. 

Likewise, an object that's very close can be modeled using very small grid cells. Using small grid cells is normally costly to performance, but that's okay if we only consider adjacent grid cells. 

So why not use both sizes? As a matter of fact, we can have a number of different cell sizes, each nested within one another. It becomes very easy this way to scale up our simulation. Let's say we have a series of nested grid cells. Each cell has a number of subdivisions, and each subdivision is half the width of its parent. It only takes about [200](http://www.wolframalpha.com/input/?i=log2+%28+%28diameter+of+the+universe%29+%2F+%28planck+length%29+%29) such subdivisions before we can simulate the entire observable universe down to the resolution of a planck unit. Assuming we represent our grid with a sparsely populated hash table, we can easily implement this on modern hardware. 

Naturally, our universe has to be very sparsely populated, but the simulation scales well with object count, too, so we can still simulate an impressive number of objects. It's totally possible to have a few million particles in our simulation, assuming we don't mind the lag. 

The algorithm itself remains fairly simple. For each object we iterate through our nested grids and examine the grid cells that are adjacent to the one housing the object. Each grid cell gets a value assigned to it, and this value represents the force exerted on every object within the grid cell. The simulator now looks something like this:

	for every object in the universe:
		for every level in the nested grid:
			for every neighboring grid cell within the level:
				calculate the force between the object and the midpoint of the grid cell

And if we want to retrieve the value at a specific location, we do the following:

	for every level in the nested grid:
		find the grid cell that intersects with our location and add its value to a running total

The solution here is known as the [fast multipole method](https://en.wikipedia.org/wiki/Fast_multipole_method). Our runtime scales linearly with the number of objects. It also scales logarithmically with the size of the simulation. Complexity is O(N log(M)), where N is the object count and and M is the simulation size. This is exactly what we want. 

## How do I use it?

Let's say you want to simulate the solar system in 2D. For starters, you'll want something to represent your gravity field: 

	field = FMM.VectorField2(resolution, range, value_function);

`resolution` expresses the smallest distance considered by the model. It is the distance at which two particles become treated as one. `range` expresses the maximum distance considered by the model. It is the distance at which two particles can no longer interact with one another.

What's `value_function`, you ask? `value_function` specifies the value at each point in the field. It accepts a 2D vector indicating the distance to a particle, and returns a 2D vector indicating the value at that distance. It can also accept an optional parameter expressing the properties of a particle, such as mass or charge. Here's what `value_function` looks like in our gravity simulator:

	function value_function(offset, particle) { 
		var distance = Math.sqrt( Math.pow(offset.x, 2) + Math.pow(offset.y, 2) );
		var acceleration = gravitational_constant * particle.mass / Math.pow(distance, 2)
		var normalized = {
			x: offset.x / distance,
			y: offset.y / distance,
		};
		return {
			x: normalized.x * acceleration
			y: normalized.y * acceleration,
		}
	}

Now we add objects to our simulation. 

	field.add_particle([0,0], { mass: 1 * solar_mass });

`add_particle()` accepts two parameters. The first expresses the location of the particle. You can use either an array of size two (e.g. `[0,0]`) or an object with `x` and `y` attributes (e.g. `{x: 0, y: 0}`). The second parameter is optional, and specifies any additional properties of the particle. 

We are now ready to retrieve values from our gravity field. 

	acceleration = field.value([1,1]);

As with `add_particle()`, the `value()` method accepts a single parameter representing location. This can either be an array of size two (`[1,1]`) or an object with `x` and `y` attributes (e.g. `{x: 1, y: 1}`).

## Other fields

You can create 2D scalar fields with a second class called `ScalarField2`.

	scalarfield = FMM.ScalarField2(resolution, range, scalar_value_function);
	
There are also equivalent classes for representing 3D scalar fields, `ScalarField3` and `VectorField3`

	scalarfield = FMM.ScalarField3(resolution, range, scalar_value_function);
	vectorfield = FMM.VectorField3(resolution, range, vector_value_function);
	
If you do any work with [Three.js](http://threejs.org/), you can also try the equivalent classes available under the `THREE` namespace. These classes operate in the exact same manner, but utilize the `THREE.Vector2` and `THREE.Vector3` classes that are provided by Three.js.

	vectorfield = THREE.VectorField2(resolution, range, vector2_value_function);
	vectorfield = THREE.VectorField3(resolution, range, vector3_value_function);
	
Lastly, if you want to work with another type of class for vectors/scalars, you can try the generic `Field2` and `Field3` classes. These generics expose two additional function parameters, `add_function` and `remove_function`, which tell the library how to add or subtract the values within the field. Here's how `THREE.VectorField3` is implemented using `Field3`:

	THREE.VectorField3 = function (resolution, range, value_fn) {
		return FMM.Field3(resolution, range, value_fn, 
			function(u, v) {
					return THREE.Vector3.addVectors( u, v );
				},
			function(u, v) {
					return THREE.Vector3.subVectors( u, v );
			 	}
		);	
	}
