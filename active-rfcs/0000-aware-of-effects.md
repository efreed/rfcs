- Start Date: 2021-06-26
- Target Major Version: 3.x
- Reference Issues: https://github.com/vuejs/vue-next/issues/617
- Implementation PR: 

# Summary

Expose a count of effects that will run if a value is modified

Whenever a Ref or Reactive object is created and managed outside of a component, then when the component is destroyed, the object still exists but a change to its value will have no effect.  When that component comes back into view, the same Ref or Reactive object is now being tracked again.

# Basic example

This is a minimal example of a ref that has an effect being added and removed from it.  The value of the ref is conditionally rendered to the DOM using v-if and the value of the ref can be modified from the console using r.value

```
<template>
	<div>
		<input type="checkbox" v-model="r_is_visible">
		<div v-if="r_is_visible">
			R = {{ r }}
		</div>
	</div>
</template>

<script setup>
	import { ref } from 'vue'

	// Ability to render or not render the value of "r"
	const r_is_visible = ref(false)

	// Simplified external storage of a Ref
	let r = window.r || (window.r = ref(10))
</script>
```

# Motivation

While Vue mananges the all the reactive interactions between a local data object and the rendered DOM, there is still work to be done to sync that data with an external source such as an API.  Development of 3rd party tools that can synchronize data between a back-end system and the local data model is a natural progression for modern web apps.

When developing a 3rd party SDK, a sophisitated one may want to be the store of the reactive data and manage automatic updating of data with thier service.  The desire is to not require any extra requirements on the developer than simply importing the reactive element and using it in thier component.

But keeping data in sync can be costly and it would be helpful to prioritize which values are still being used by the UI.  For example, update the ones that are in use in real time, while the others could be put off for later.

There are no alternatives that can be implemented in the user space (official, or unofficial).  Without adding this feeature (either officially or unofficially) then Vue3 will have less functionality than Vue2.  See here for an example of how it is possible in Vue2: https://stackoverflow.com/questions/68131994/how-to-know-if-a-vue3-reactive-object-is-being-tracked

# Detailed design

Add this function into Vue3 which will return the number of effects that will run if a value is modified or triggered.

```
/**
 * Returns number of effects that will run if this target's key is modified
 * If target is a ref, then key is optional
 * @param object target
 * @param string key
 * @returns int
 */
function triggerCount(target, key) {
	const depsMap = targetMap.get(toRaw(target));
	if (!depsMap) {
		return 0;
	}
	const dep = depsMap.get(key || 'value');
	return dep ? dep.size : 0;
}
```

This will allow an SDK to decide if it wants to keep syncronizing data, based on if the value is zero or not.  (Or possibly the SDK implemented its own watchEffect that it is aware of and should not count, so it can choose to put the threshold at 1 or 2)

In this use case, it is not an urgent thing to know when a reactive element *stops* being rendered to the DOM, it is something that could be checked before an expensive operation is performed.

To know when a value is being rendered to the DOM again (or has some other effect), it would be quite simple for an SDK to wrap the reactive objects in a way that provides an immediate notification when it is used.  To spell that out, here's an update to the example given above:

```
<template>
	<div>
		<input type="checkbox" v-model="r_is_visible">
		<div v-if="r_is_visible">
			R = {{ sdk.r }}
		</div>
	</div>
</template>

<script setup>
	import { reactive, ref } from 'vue'

	// Ability to render or not render the value of "r"
	const r_is_visible = ref(false)

	// Simplified external storage of a Ref
	const sdk = sample_sdk();

	// Begin 3rd party code
	function sample_sdk() {
        const data = window.sdkData || (window.sdkData = reactive({r:10}));

		function expensiveTask() {
			if (triggerCount(data,"r") > 0) {
				// Worth it
			} else {
				// Wait until it gets read again
			}
		}

		return new Proxy(data, {
			get: function(target, prop) {
				console.log(prop + " was read so if it had no triggerCount before, check again!");
				return target[prop];
			}
		});
	}
</script>
```

# Drawbacks

Implementation cost of a small, independent function is a light non-breaking change.  But I don't have a 'cost' estimate for updating documenation and tests.  There is surely future compatibility cost whenever a functionality is offered, so this recommended solution was designed for simplicity and the minimal amount of support in the Vue core library.

# Alternatives

- Maybe this is over simplified and deserves a full event system.  There is no "event" model in Vue3 other than watchers, so hard to recommend a specific architecture.  If watchEffect was used, we'd need need something that would get called inside the watch function that knows which reactive properties we care about so when effects are added or removed the watch function will run again.  Which sure sounds a lot like a triggerCount() function.  So a triggerCount function is still the first step in this direction and a full event model would be provided if watchEffect were to recognize usages of triggerCount()
- Maybe there's a existing planned feature that would accomplish this goal
- Instead of a new function name, it could be integrated into an existing one.  trigger() for example could have some "dry run" option or extra parameter to make the function return the count of how many functions it would have triggered instead of actually triggering them.  There's probably a code design rule this breaks that functions should not return different types of data based on parameters.
- Simply expose the private targetMap property with a getTargetMap() function.  This would save a few lines of code, does not require any tests, and doesn't promise future support for any specific value of functionality.  But it would enable more undocumented uses of this data than is required.

# Adoption strategy

This is additional functionality, so will not effect any existing Vue code or developer training.

# Unresolved questions

- The name of the function
- What level of documentation is needed, since the main audience is SDK developers.  Should it be something new devlopers are made aware of, or listed in an "advanced" section?
