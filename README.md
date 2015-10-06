# fast-multipole-method.js
This is a javascript implementation for the [fast multipole method](https://en.wikipedia.org/wiki/Fast_multipole_method)

## The problem
Let's say you want to simulate a solar system. You'll recall all objects in the universe effect one another through gravity, right? A naïve simulator might simulate this as follows:

    for every object in the universe:
    	for every other object in the universe:
    		calculate the force exerted between the objects

The simulation above doesn't scale very well as we increase the number of objects. If I want to simulate 3 objects I have to make 6 calculations per timestep. If I want to simulate 4 objects I have to make 12 calculations per timestep - I doubled the number of calculations just by adding one object! Things get out of hand quickly. What I really want is for the number of calculations to increase *in direct proportion* to the number of objects.

Well, fine then, why don't I do it this way:

    for every object in the universe:
    	for every location in the universe:
    		calculate the force exerted by the object from that location and store it in a grid somewhere
    
The "grid" we mention is an approximation for what physicists call a [field](https://en.wikipedia.org/wiki/Field_(physics)). For every point on a field, there is a corresponding value. Sometimes, that value is a scalar, like a floating point number. Sometimes, that value is a vector, which might express something like force or acceleration. Gravity, for instance, can be represented by a field of vectors, each vector expressing the acceleration one would experience at a specific location.

Now it's true the solution above scales well with the number of objects, but it does not scale well with the size of the grid. If I want to simulate a universe that's 3 grid cells wide I have to make 9 calculations for each object. If I want to simulate a universe that's 4 grid cells wide I have to make 16 calculations for each object. Again, things get out of hand quickly. What we really need is a simulation that scales well for both the number of objects *and* the size of the grid.

There is another thing we could do. We could just not consider the effects of gravity past a certain distance. A grid also works well for this implementation - if two objects occupy adjacent grid cells, we can assume the objects are within range. There's still a few problems with this approach, though. It doesn't work well for really massive objects, obviously. Also, if too many objects occupy the same grid cell, we're back to same problem we had in our first solution.

## The fast multipole method
The solution to our problem relies upon an observation. As you move further away from an object, it becomes less important to model the force of gravity accurately. This is because gravity follows an [inverse square law](https://en.wikipedia.org/wiki/Inverse-square_law). The further you go, the less gravity you feel, and the less important it becomes to get the details. The only thing that continues to matter is its order of magnitude. 

So what if we had several grids, each with different cell size? As with our previous model, we'd only consider gravity within adjacent grid cells. However, because the grid cells come in different sizes, we can represent fields from any object, no matter how massive. And because gravity diminishes with distance, we don't have to worry about accuracy when the grid cells are large. 

We can make these grid cells any size we want. If cell size scales geometrically, there's no limit to the size of our simulation, provided our grid is represented by a sparsely populated hash table. Let's say we have a series of nested grid cells where each cell is composed of two smaller subdivisions. It only takes [200](http://www.wolframalpha.com/input/?i=log2+%28+%28diameter+of+the+universe%29+%2F+%28planck+length%29+%29) such subdivisions before we can simulate the entire observable universe down to the resolution of a planck unit. This, of course, assumes our universe is very sparsely populated, perhaps by no more than a few million particles. Still, we've made our point. 

Our new algorithm remains simple. For each object we look at the grid cells adjacent to the one that houses the object. Each grid cell gets a value assigned to it, and this value represents the force exerted on every object within the grid cell. The simulator now looks something like this:

	for every object in the universe:
		for every level in the nested grid:
			for every neighboring grid cell within the level:
				calculate the force between the object and the midpoint of the grid cell

This is known as the [fast multipole method](https://en.wikipedia.org/wiki/Fast_multipole_method). Our runtime scales linearly with the number of objects. It also scales logarithmically with the size of the simulation. Complexity is O(N log(M)), where N is the object count and and M is the simulation size. This is exactly what we want. 

## How do I use it?

Let's say you want simulate the solar system in 2D. For starters, you'll want something to represent your gravity field: 

	field = FMM.VectorField2(resolution, range, value_function);

`resolution` expresses the smallest distance considered by the model. It is the distance at which two particles become treated as one. `range` expresses the maximum distance considered by the model. It is the distance at which two particles can no longer interact with one another.

What's `value_function`, you ask? `value_function` specifies the value at each point in the field. It accepts a 2D vector indicating the distance to a particle, and returns a 2D vector indicating the value. It can also accept an optional parameter expressing the properties of a particle, such as mass or charge. Here's what `value_function` looks like in our gravity simulator:

	function value_function(offset, particle) { 
		var distance = Math.sqrt( Math.pow(offset.x, 2) + Math.pow(offset.y, 2) );
		var normalized = {
			x: offset.x / distance,
			y: offset.y / distance,
		};
		var acceleration = {
			x: normalized.x * gravitational_constant * particle.mass / Math.pow(distance, 2),
			y: normalized.y * gravitational_constant * particle.mass / Math.pow(distance, 2),
		}
		return acceleration;
	}

Now we add objects to our simulation. 

	field.add_particle([0,0], { mass: 1 * solar_mass });

`add_particle()` accepts two parameters. The first expresses the location of the particle. You can use either an array of size two (e.g. `[0,0]`) or an object with `x` and `y` attributes (e.g. `{x: 0, y: 0}`). The second parameter is optional, and specifies any additional properties of the particle. 

We are now ready to retrieve values from our gravity field. 

	acceleration = field.value([1,1]);

As with `add_particle()`, the `value()` method accepts a single parameter representing location. This can either be an array of size two (`[1,1]`) or an object with `x` and `y` attributes (e.g. `{x: 1, y: 1}`).
