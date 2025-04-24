---
title: Bevy 渲染图结构
date: 2025-04-24
updated: 2025-04-24
categories: Bevy
tags:
  - Bevy
  - 渲染
top: 1
---

## Bevy 渲染图结构介绍

**`RenderGraph`** 是`bevy`用来处理渲染工作负载的任务调度器，是由`Nodes`和`Edges`（节点和边）连接成一个有向无环计算图。
每个节点代表一个执行渲染工作的独立单元（一个任务），边表示节点间的依赖关系（执行顺序）。

### Node

- 节点由`RenderGraph`调用，具体是由`RenderGraphRunner`调用`run_graph`运行节点，同时也会运行节点的`SubGraph`。[查看源码](https://github.com/bevyengine/bevy/blob/12f71a8936c77735b6f702b1ef849dfc8a306a37/crates/bevy_render/src/renderer/graph_runner.rs#L231)
- `RenderGraph`会由`render_system`调用。[查看源码](https://github.com/bevyengine/bevy/blob/12f71a8936c77735b6f702b1ef849dfc8a306a37/crates/bevy_render/src/renderer/mod.rs#L42)


节点的`trait`定义:

```Rust

pub trait Node: Downcast + Send + Sync + 'static {
    /// Specifies the required input slots for this node.
    /// They will then be available during the run method inside the [`RenderGraphContext`].
    fn input(&self) -> Vec<SlotInfo> {
        Vec::new()
    }

    /// Specifies the produced output slots for this node.
    /// They can then be passed one inside [`RenderGraphContext`] during the run method.
    fn output(&self) -> Vec<SlotInfo> {
        Vec::new()
    }

    /// Updates internal node state using the current render [`World`] prior to the run method.
    fn update(&mut self, _world: &mut World) {}

    /// 运行图节点逻辑，发出draw调用，更新输出槽
    /// 并可选地将子图排队执行。图形数据、输入和输出值通过RenderGraphContext传递。
    fn run<'w>(
        &self,
        graph: &mut RenderGraphContext,
        render_context: &mut RenderContext<'w>,
        world: &'w World,
    ) -> Result<(), NodeRunError>;
}

```

`Node::run` 提供了节点不可变引用，`RenderGraphContext`和`RenderContext`可变引用，以及`World`不可变引用。

1. `RenderGraphContext`主要用于运行`SubGraph`(`run_sub_graph()`)。其具有的插槽功能，不必理会，源代码中也只有test用例。
    - 参考`CameraDriverNode`，[查看源码](https://github.com/bevyengine/bevy/blob/12f71a8936c77735b6f702b1ef849dfc8a306a37/crates/bevy_render/src/camera/camera_driver_node.rs#L56)
2. `RenderContext` 能够获取用于渲染的`GPU`实例，如`CommandEncoder`,`RenderDevice`, `TrackedRenderPass`等实现调用`GPU`进行绘制图形。
    - 示例：[MainOpaquePass2dNode](https://github.com/bevyengine/bevy/blob/12f71a8936c77735b6f702b1ef849dfc8a306a37/crates/bevy_core_pipeline/src/core_2d/main_opaque_pass_2d_node.rs#L21)、[MainTransparentPass2dNode](https://github.com/bevyengine/bevy/blob/12f71a8936c77735b6f702b1ef849dfc8a306a37/crates/bevy_core_pipeline/src/core_2d/main_transparent_pass_2d_node.rs#L17)。
    ```Rust
    render_context.add_command_buffer_generation_task(move |render_device|{
        // Command encoder setup
        let mut command_encoder =
            render_device.create_command_encoder(&CommandEncoderDescriptor {
                label: Some("main_opaque_pass_2d_command_encoder"),
        });

        // Render pass setup
        let render_pass = command_encoder.begin_render_pass(&RenderPassDescriptor {
            label: Some("main_opaque_pass_2d"),
            color_attachments: &color_attachments,
            depth_stencil_attachment,
            timestamp_writes: None,
            occlusion_query_set: None,
        });
        let mut render_pass = TrackedRenderPass::new(&render_device, render_pass);
        // ...该位置会由RenderPhase 调用render函数，这里两种节点处理方式不一样
        pass_span.end(&mut render_pass);
        drop(render_pass);
        command_encoder.finish()
    });
    ```
3. `World`能够获取所有注册在`RenderWorld`中的`Resource`。作为渲染所需要的资源。比如`PipelineCache`等资源，
    - 示例：
    ```Rust
    let pipeline_cache = world.resource::<PipelineCache>();
    // Or
    let Some(pipeline_cache) = world.get_resource::<PipelineCache>() else {
        return Ok(());
    }
    ```

节点也能够查询`Component`（经过`Extract`阶段提取到渲染世界中的实体）

- 示例：依旧参考`CameraDriverNode`
    ```Rust
        pub struct CameraDriverNode {
            cameras: QueryState<&'static ExtractedCamera>,
        }
    ```
    此时需要特化`Node`的`update`方法。
    ```Rust
    impl Node for CameraDriverNode {
        fn update(&mut self, world: &mut World) {
            self.cameras.update_archetypes(world);
        }
    }
    ```

### ViewNode

实现该`trait`的节点由`ViewNodeRunner`执行，`ViewNodeRunner`是一个泛型结构体，因此它可以被不同的实现特化。
`ViewNodeRunner`实现了`Node`trait。[查看源码](https://github.com/bevyengine/bevy/blob/12f71a8936c77735b6f702b1ef849dfc8a306a37/crates/bevy_render/src/render_graph/node.rs#L381)

```Rust
impl<T> Node for ViewNodeRunner<T>
where
    T: ViewNode + Send + Sync + 'static,
{
    fn update(&mut self, world: &mut World) {
        self.view_query.update_archetypes(world);
        self.node.update(world);
    }

    fn run<'w>(
        &self,
        graph: &mut RenderGraphContext,
        render_context: &mut RenderContext<'w>,
        world: &'w World,
    ) -> Result<(), NodeRunError> {
        let Ok(view) = self.view_query.get_manual(world, graph.view_entity()) else {
            return Ok(());
        };

        ViewNode::run(&self.node, graph, render_context, view, world)?;
        Ok(())
    }
}
```

减少了`Node`获取`Component`和更新`QueryState`的模板代码，同时又能实现运行属于不同的`Camera`的`ViewEntities`。
在`CameraDriverNode`中`中ViewNode`节点只负责渲染当前`Camera`中内容。
多个`Camera`由`SortCamera`（已经排好序）在`CameraDriverNode`的`run`函数中一次执行，运行每一个`Camera`下的`SubGraph`。
