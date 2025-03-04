# Custom Models

The _Adapters_ library provides a simple mechanism for integrating adapter methods into any available _Transformers_ model - including custom architectures.
This can be accomplished by defining a plugin interface instance of [`AdapterModelInterface`](adapters.AdapterModelInterface).
The following example shows how this looks like for Gemma 2:

```python
import adapters
from adapters import AdapterModelInterface
from transformers import AutoModelForCausalLM

plugin_interface = AdapterModelInterface(
    adapter_methods=["lora", "reft"],
    model_embeddings="embed_tokens",
    model_layers="layers",
    layer_self_attn="self_attn",
    layer_cross_attn=None,
    attn_k_proj="k_proj",
    attn_q_proj="q_proj",
    attn_v_proj="v_proj",
    attn_o_proj="o_proj",
    layer_intermediate_proj="mlp.up_proj",
    layer_output_proj="mlp.down_proj",
)

model = AutoModelForCausalLM.from_pretrained("google/gemma-2-2b-it", token="<YOUR_TOKEN>")
adapters.init(model, interface=plugin_interface)

model.add_adapter("my_adapter", config="lora")

print(model.adapter_summary())
```

## Walkthrough

Let's go through what happens in the example above step by step:

**1. Define adapter methods to plug into a model:**  
The `adapter_methods` argument is the central parameter to configure which adapters will be supported in the model.
Here, we enable all LoRA and ReFT based adapters.
See [`AdapterMethod`](adapters.AdapterMethod) for valid options to specify here.
Check out [Adapter Methods](methods.md) for detailed explanation of the methods.

**2. Define layer and module names:**  
While all Transformers layers share similar basic components, their implementation can differ in terms of subtleties such as module names.
Therefore, the [`AdapterModelInterface`](adapters.AdapterModelInterface) needs to translate the model-specific module structure into a common set of access points for adapter implementations to hook in.
The remaining attributes in the definition above serve this purpose.
Their attribute names follow a common syntax that specify their location and purpose:
- The initial part before the first "_" defines the base module relative to which the name should be specified.
- The remaining part after the first "_" defines the functional component.

E.g., `model_embeddings` identifies the embeddings layer (functional component) relative to the base model (location).
`layer_output_proj` identifies the FFN output projection relative to one Transformer layer.
Each attribute value may specify a direct submodule of the reference module (`"embed_token"`) or a multi-level path starting at the reference module (`"mlp.down_proj"`).

**3. (optional) Extended interface attributes:**  
There are a couple of attributes in the [`AdapterModelInterface`](adapters.AdapterModelInterface) that are only required for some adapter methods.
We don't need those in the above example for LoRA and ReFT, but when supporting bottleneck adapters as well, the full interface would look as follows:
```python
adapter_interface = AdapterModelInterface(
    adapter_types=["bottleneck", "lora", "reft"],
    model_embeddings="embed_tokens",
    model_layers="layers",
    layer_self_attn="self_attn",
    layer_cross_attn=None,
    attn_k_proj="k_proj",
    attn_q_proj="q_proj",
    attn_v_proj="v_proj",
    attn_o_proj="o_proj",
    layer_intermediate_proj="mlp.up_proj",
    layer_output_proj="mlp.down_proj",
    layer_pre_self_attn="input_layernorm",
    layer_pre_cross_attn=None,
    layer_pre_ffn="pre_feedforward_layernorm",
    layer_ln_1="post_attention_layernorm",
    layer_ln_2="post_feedforward_layernorm",
)
```

**4. Initialize adapter methods in the model:**
Finally, we just need to apply the defined adapter integration in the target model.
This can be achieved using the usual `adapters.init()` method:
```python
adapters.init(model, interface=adapter_interface)
```
Now, you can use (almost) all functionality of the _Adapters_ library on the adapted model instance!

## Limitations

The following features of the _Adapters_ library are not supported via the plugin interface approach:
- Prefix Tuning adapters
- Parallel composition blocks
- XAdapterModel classes
- Setting `original_ln_after=False` in bottleneck adapter configurations (this affects `AdapterPlusConfig`)
