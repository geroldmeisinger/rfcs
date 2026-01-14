# RFC: Execution mode in sequential processing

- Start Date: 2026-01-14
- Target Major Version: 0.10
- Reference Issues: https://github.com/Comfy-Org/ComfyUI/issues/11131
- Implementation PR: (leave this empty)

## Summary

ComfyUI provides [data lists](https://docs.comfy.org/custom-nodes/backend/lists) which cause nodes to process multiple times. This is very useful for workflows which make use of multiple assets (images, prompts, etc.). However, the "multiple processing of a single node" stalls the workflow on the node and ComfyUI should pass on execution to the next node after each result and provide a way to control execution mode in sequential processing.

## Motivation

> Why are we doing this? What use cases does it support? What is the expected outcome?
> Please focus on explaining the motivation so that if this RFC is not accepted, the motivation could be used to develop alternative solutions. In other words, enumerate the constraints you are trying to solve without coupling them too closely to the solution you have in mind.

### Simple example

In the following example, the node `String OutputList` is marked as `OUTPUT_IS_LIST=True` (a data list) and tells `KSampler` to fetch and process the items sequentially:

https://github.com/user-attachments/assets/6bb38b4c-f00e-4636-8d23-277be547de18

This works well for small workflows but with larger and more advanced workflows the execution remains at one node for a very long time and also potentially builds up memory usage.

### XYZ-GridPlots

We often want to make image grids to compare how different parameters effect generation.

<img width="2890" height="820" alt="Image" src="https://github.com/user-attachments/assets/2a8d9686-b780-45e3-b38b-13ab791d776c" />

In this example the `KSampler` processes ALL combinations before the next nodes (`VAE Decode > Save Image`) which can take a very long time and you could loose all progress if something happens. Instead, we want to see and save the intermediate results of each image immediately.

### Iterate models

We often want to iterate over a list of models to compare how different checkpoints or LoRAs effect generation.

<img width="2350" height="736" alt="Image" src="https://github.com/user-attachments/assets/8a5b85b6-cf54-40e1-8336-ce20c71d34c1" />

In this example the node `Load Checkpoint` will load ALL checkpoints first before proceeding which eventually leads to OOM.

### Bake parameters into workflow

We also want to bake parameters into the workflow JSON from a multi-image generating workflow to see which concrete parameter was used to generate a specific image. Otherwise, every image stores the same (multi-image generating) workflow.

<img width="1360" height="292" alt="Image" src="https://github.com/user-attachments/assets/758c72a7-8928-404e-9de4-b245a968e632" />

In this example the node `FormattedString` makes use of `extra_pnginfo` in the second (dynprompt) textfield to store the actually used prompt in the workflow. But this doesn't work because the node is processed for all items first and only stores the last entry before eventually proceeding to `SaveImage`. When you load any workflow from all the images they all only show the last entry.

### Multi-model workflows

Another usecase to consider are multi-model workflows. Let's say we have a list of prompts and want to process them in a powerful "generate image -> generate video -> upscale video" pipeline, but only one step will ever fit into VRAM. Thus we want to do something like "load image model -> generate ALL images -> unload image model -> load video model -> generate ALL videos -> unload video model -> upscale ALL videos". So in this case we need some control over how the multi-processing should work.

## Detailed design

> This is the bulk of the RFC. Explain the design in enough detail for somebody familiar with Vue to understand, and for somebody familiar with the implementation to implement. This should get into specifics and corner-cases, and include examples of how the feature is used. Any new terminology should be defined here.

I'm not a core developer and cannot provide implementation details, only describe how it would appear to the user. Please note that "node expansion" already provides some features and is discussed in Alternative Solution below!

### Solution 1 - Pass on execution immediately

I would argue that all the examples except "Multi-model workflows" are the more common case and "Multi-model workflow" is very special and advanced. As such, the default behaviour should be to **pass on execution immediately instead of processing the same node multiple times at once**. Thus in the case of "multi-model workflow" a change in execution mode would only concern the `KSampler` and we need an option  "for this node only, process all items multiple times at once." (That is, unless the user also wants to see intermediate results before unloading the model.)

But whatever the default case is, there has to be a way to change execution mode, either for one node or a group of nodes.

### Solution 2 - Control execution for a single node

The other point to discuss would be - whether or not the default execution mode is actually changed - HOW we set a single node or group of nodes to the other execution mode. The single node case is trivial, just add a context menu entry and some visual that the node uses a different execution mode.

### Solution 3 - Control execution for a group of nodes

**a) Subgraphs**

Add an option for subgraph "treat as sequential unit".

<img width="806" height="304" alt="Image" src="https://github.com/user-attachments/assets/370a92bd-95df-44e3-beef-30e16132a476" />

- Pro: Subgraphs already covers the concept of groups
- Con: Requires hierachical treatment of data lists, otherwise subgraphs would work differently if they are executed as a standalone workflow. Let's say there is output list outside of the subgraph, then the whole subgraph executes as whole on this item. If there is another output list within the subgraph, the behaviour should work like data lists work now (but what if different hierarchical layers are intertwined?).

**b) Add Group (box)**

When the box is set as "sequential unit" all nodes within the box execute immediately.

<img width="648" height="340" alt="Image" src="https://github.com/user-attachments/assets/d8bef0e7-120e-4722-8438-4d75b4e76eb5" />

- Pro: easy to mark 
- Con: could lead to ambiguous situations when only half of a graph flow is marked

**c) onExecuted/onTrigger**

I don't know what the original intention for this feature was but it could be used to the tell `KSampler`: "after you have executed, trigger this `SaveImage` node immediately.. which then looks upstream until it finds the `KSampler` and fetches the item downstream. It only needs to be actually implemented.

<img width="853" height="378" alt="Image" src="https://github.com/user-attachments/assets/32447fd0-4693-479c-90ef-99c8af312657" />

- Pro: it somewhat already allows to control execution, so maybe this RFC is just an argument to finally implement it
- Con: has to be implemented first

comfyanonymous/ComfyUI#704
comfyanonymous/ComfyUI#1135
comfyanonymous/ComfyUI#7443 (I tried it here: https://github.com/comfyanonymous/ComfyUI/pull/7443#issuecomment-3592346229)

**d) Group nodes**

(Just for completeness sake, not seriously considered)

<img width="570" height="653" alt="Image" src="https://github.com/user-attachments/assets/ae897aa5-b1a6-4eb8-b3ea-54f50d14115b" />

- Pro: same as subgraph
- Con: Groups have been deprecated

## Drawbacks

> Why should we *not* do this? Please consider:

> - implementation cost, both in term of code size and complexity

I don't know. From an execution point of view it would execute the same node multiple times. So this could be effected by Muting/Bypassing and any flow-control nodes. And all output nodes which support image lists: Save Image or Preview Image would populate the list one by one (or at least save one by one if we don't care about the presentation).

> - whether the proposed feature can be implemented in user space

Yes, see "ad-hoc node expansion" but this solution is ambigious and has some drawbacks.

> - the impact on teaching people ComfyUI
> - cost of migrating existing ComfyUI applications (is it a breaking change?)

I consider the impact of teaching and migrating small, because it only applies to advanced workflows and the change is simply "from now on, nodes pass on execution immediately".

> - integration of this feature with other existing and planned features

onExecuted/onTrigger could be effected by this

> There are tradeoffs to choosing any path. Attempt to identify them here.

has been discussed

## Alternatives

> What other designs have been considered?

**a) custom node expansion**

The only solution so far I've found is to copy the node pattern `KSampler > VAE Decode > Save Image` as a custom node with node expansion in code:

<img width="2490" height="748" alt="Image" src="https://github.com/user-attachments/assets/1be7d128-8c74-4087-b106-24e378c96c7f" />

```
class KSamplerImmediateSave:
	DESCRIPTION="""Node Expansion of default KSampler, VAE Decode and Save Image to process as one.
This is useful if you want to save the intermediate images for grids immediately.
'A custom KSampler just to save an image? Now I have become the very thing I sought to destroy!'
"""
	@classmethod
	def INPUT_TYPES(cls):
		return {
			"required": {
				# from ComfyUI/nodes.py KSampler
				"model"       	: ("MODEL"       	, { "tooltip" : "The model used for denoising the input latent." } ) ,
				"positive"    	: ("CONDITIONING"	, { "tooltip" : "The conditioning describing the attributes you want to include in the image." } ) ,
				"negative"    	: ("CONDITIONING"	, { "tooltip" : "The conditioning describing the attributes you want to exclude from the image." } ) ,
				"latent_image"	: ("LATENT"      	, { "tooltip" : "The latent image to denoise." } ) ,
				"vae"         	: ("VAE"         	, { "tooltip" : "The VAE model used for decoding the latent." } ) ,

				"seed"        	: ("INT"                             	, {"default" : 	0  	, "min" :	0  	, "max" :	0xfffffffffffffff  	, "control_after_generate" : True,	"tooltip" : "The random seed used for creating the noise." } ) ,
				"steps"       	: ("INT"                             	, {"default" :	20  	, "min" :	1  	, "max" :            	10000  	,                                 	"tooltip" : "The number of steps used in the denoising process." } ) ,
				"cfg"         	: ("FLOAT"                           	, {"default" : 	8.0	, "min" :	0.0	, "max" :              	100.0	, "step" : 0.1, "round" : 0.01,   	"tooltip" : "The Classifier-Free Guidance scale balances creativity and adherence to the prompt. Higher values result in images more closely matching the prompt however too high values will negatively impact quality." } ) ,
				"sampler_name"	: (comfy.samplers.KSampler.SAMPLERS  	, {"tooltip" : "The algorithm used when sampling , this can affect the quality , speed , and style of the generated output." } ) ,
				"scheduler"   	: (comfy.samplers.KSampler.SCHEDULERS	, {"tooltip" : "The scheduler controls how noise is gradually removed to form the image." } ) ,
				"denoise"     	: ("FLOAT"                           	, {"default" :	1.0	, "min" :	0.0	, "max" :	1.0	, "step" : 0.01,	"tooltip" : "The amount of denoising applied , lower values will maintain the structure of the initial image allowing for image to image sampling." } ) ,

				# from ComfyUI/nodes.py SaveImage
				"filename_prefix"	: ("STRING", {"default" : "ComfyUI", "tooltip" : "The prefix for the file to save. This may include formatting information such as %date :yyyy-MM-dd% or %Empty Latent Image.width% to include values from nodes."}),
			},
			# "hidden": {
			#	"prompt": "PROMPT", "extra_pnginfo": "EXTRA_PNGINFO",
			# },
		}

	RETURN_NAMES   	= ("image", )
	RETURN_TYPES   	= ("IMAGE", )
	OUTPUT_TOOLTIPS	= ("The decoded image.",) # from ComfyUI/nodes.py VAEDecode
	OUTPUT_NODE    	= True
	FUNCTION       	= "execute"
	CATEGORY       	= "_for_testing"

	def execute(self, model, positive, negative, latent_image, vae, seed, steps, cfg, sampler_name, scheduler, denoise, filename_prefix):
		graph 	= GraphBuilder()
		latent	= graph.node("KSampler" , model=model, positive=positive, negative=negative, latent_image=latent_image, seed=seed, steps=steps, cfg=cfg, sampler_name=sampler_name, scheduler=scheduler, denoise=denoise)
		images	= graph.node("VAEDecode", samples=latent.out(0), vae=vae)
		save  	= graph.node("SaveImage", images=images.out(0), filename_prefix=filename_prefix)
		return {
			"result" : (images.out(0),),
			"expand" : graph.finalize(),
		}
```

While this works with the default pattern in the simple example, it would require to reimplement each deviation as a new custom node in code.
* You want to use two samplers? New custom node
* You want to apply some image manipulation before saving? New custom node
* You want to include the Load Checkpoint for model iteration? New custom node
etc.

**b) ad-hoc node expansion in editor**

Because node expansion already works we could provide a feature "create node-expansion from selected nodes" which generates the node expansion code and just provides a custom node which is only valid within this workflow.

**c) ad-hoc node expansion in runtime**

Another solution could be to create an ad-hoc node expansion based on the connected nodes (the following is just an draft, I didn't implement this for real):

<img width="1056" height="437" alt="Image" src="https://github.com/user-attachments/assets/c9828277-3479-40fd-8fbd-403f263d0b2a" />

`SequentialUnitStart` is located between the OutputList and the `KSampler` and just passes the item through. `SequentialUnitEnd` is located after the output node `SaveImage` and passes the value trough downstream. When the prompt is executed, look up the connected starting node (`KSampler`), the connected end node (`SaveImage`) and find the linked graph and nodes in-between, copy all the nodes as a node expansion, set the original nodes as "muted", and execute the original behaviour on the node expansion.

Cons:
1. this is very tedious to setup as a user.
2. it is not clear were the starting node should be (right after the `OUTPUT_IS_LIST=True` (here: `String OutputList`), or is it okay if it's further downstream (here: `TextEncode`). what if there are multiple output lists connected to the `KSampler`?
3. `SaveImage` only has an input knob, so `SequentialUnitEnd` needs to connect to the input, but then no else can connect to it. If `SequentialUnitEnd` is connected before `SaveImage` then intuitively it looks as if `SaveImage` is excluded from the sequential unit. If it should look as if it's inside the unit, it must provide an output knob, which `SaveImage` does not (hence the custom `SaveImage2` in this example).
4. we could forego linking completely and just work with node ids. but this is even more tedious, error prone and makes no sense in ComfyUI.

**d) The PrimitiveInt control_after_generate=increment pattern**

A well known pattern to emulate multi processing is the `Primitive Int` node with `control_after_generate=increment`. You basically get a counter that increases everytime you run a prompt. When you hook it up as a index in a list selector node, it iterates over entries across multiple prompts. In the Run toolbox you can set the amount of prompts to the number of items in your list to iterate the whole list. This pattern essentially cancels out the effect of output lists and will only ever process one item at a time.

<img width="1056" height="437" alt="Image" src="https://raw.githubusercontent.com/geroldmeisinger/ComfyUI-outputlists-combiner/main/media/PrimitiveIntControlAfterGenerateIncrement.png" />


> What is the impact of not doing this?

For any serious multi-image workflows the current sequential processing always runs into road blocks. Either we have to wait a long time to see intermediate results, where the progress could be lost, or it runs into OOM.

## Adoption strategy

> If we implement this proposal, how will existing ComfyUI users and developers adopt it? Is this a breaking change? How will this affect other projects in the ComfyUI ecosystem?

I don't see data lists wildly used right now although they should be as they are very powerful. As such I think breaking changes are minimal and only effect multi-image-multi-model workflows with model unloading (see examples above), which are very unlikely to exist.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still TBD?

