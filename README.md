


import bpy

# Текстовая переменная для хранения имени текущего выделенного объекта или коллекции
current_selection_name = ""

def switch_text_block(context):
    global current_selection_name
    
    # Получаем список выбранных объектов и коллекций
    selected_objects = context.selected_objects
    
    if selected_objects:
        current_selection_name = selected_objects[0].name
    else:
        # Проверяем активную коллекцию в аутлайнере
        active_collection = bpy.context.view_layer.active_layer_collection.collection
        if active_collection:
            current_selection_name = active_collection.name
        else:
            current_selection_name = ""
    
    text_name = f"{current_selection_name}_text_block"
    text_block = bpy.data.texts.get(text_name)
    
    # Открыть текстовый блок или закрыть текущий, если его нет
    for area in context.screen.areas:
        if area.type == 'TEXT_EDITOR':
            for space in area.spaces:
                if space.type == 'TEXT_EDITOR':
                    if text_block:
                        space.text = text_block
                    else:
                        space.text = None

class TEXT_BLOCK_OT_create_or_open(bpy.types.Operator):
    bl_idname = "text_block.create_or_open"
    bl_label = "Create Text Block"

    def execute(self, context):
        global current_selection_name
        
        if not current_selection_name:
            self.report({'WARNING'}, 'No active object or collection')
            return {'CANCELLED'}
        
        text_name = f"{current_selection_name}_text_block"
        text_block = bpy.data.texts.get(text_name)
        
        if not text_block:
            text_block = bpy.data.texts.new(name=text_name)

        switch_text_block(context)
        return {'FINISHED'}

class TEXT_BLOCK_OT_delete_text(bpy.types.Operator):
    bl_idname = "text_block.delete_text"
    bl_label = "Delete Text Block"

    def execute(self, context):
        global current_selection_name
        
        if not current_selection_name:
            self.report({'WARNING'}, 'No active object or collection')
            return {'CANCELLED'}
        
        text_name = f"{current_selection_name}_text_block"
        text_block = bpy.data.texts.get(text_name)
        
        if text_block:
            bpy.data.texts.remove(text_block)

        switch_text_block(context)
        return {'FINISHED'}

class TEXT_BLOCK_OT_cleanup_texts(bpy.types.Operator):
    bl_idname = "text_block.cleanup_texts"
    bl_label = "Cleanup Unused Text Blocks"

    def execute(self, context):
        objects_and_collections_names = {obj.name for obj in bpy.data.objects} | {col.name for col in bpy.data.collections}
        
        # Подсчет количества привязанных и непривязанных текстовых блоков
        bound_texts_count = 0
        unbound_texts_count = 0
        
        for text in bpy.data.texts:
            if text.name == "Script":
                continue  # Пропускаем текстовый блок с названием "Script"
            
            if any(text.name.startswith(name) for name in objects_and_collections_names):
                bound_texts_count += 1
            else:
                unbound_texts_count += 1
                bpy.data.texts.remove(text)
        
        self.report({'INFO'}, f"Cleaned up {unbound_texts_count} unused text blocks. {bound_texts_count} bound text blocks remain.")
        return {'FINISHED'}

class TEXT_BLOCK_PT_panel(bpy.types.Panel):
    bl_label = "Open Text Manager"
    bl_idname = "OBJECT_PT_text_block_manager"
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = 'Tools'

    def draw(self, context):
        layout = self.layout
        global current_selection_name
        
        if not current_selection_name:
            layout.label(text="No active object or collection", icon='INFO')
        else:
            text_name = f"{current_selection_name}_text_block"
            text_block = bpy.data.texts.get(text_name)

            row = layout.row()
            if not text_block:
                row.operator(TEXT_BLOCK_OT_create_or_open.bl_idname, text=f"Create {current_selection_name} Text Block")

            row = layout.row()
            if text_block:
                row.operator(TEXT_BLOCK_OT_delete_text.bl_idname, text="Delete Text Block")
        
        # Подсчет количества привязанных и непривязанных текстовых блоков
        objects_and_collections_names = {obj.name for obj in bpy.data.objects} | {col.name for col in bpy.data.collections}
        bound_texts_count = 0
        unbound_texts_count = 0
        
        for text in bpy.data.texts:
            if text.name == "Script":
                continue  # Пропускаем текстовый блок с названием "Script"
            
            if any(text.name.startswith(name) for name in objects_and_collections_names):
                bound_texts_count += 1
            else:
                unbound_texts_count += 1
        
        cleanup_text = f"Cleanup ({unbound_texts_count} unused, {bound_texts_count} bound)"
        
        row = layout.row()
        row.operator(TEXT_BLOCK_OT_cleanup_texts.bl_idname, text=cleanup_text)

def on_selection_change(context):
    switch_text_block(context)

def depsgraph_update_post(scene):
    context = bpy.context
    on_selection_change(context)

def register():
    bpy.utils.register_class(TEXT_BLOCK_OT_create_or_open)
    bpy.utils.register_class(TEXT_BLOCK_OT_delete_text)
    bpy.utils.register_class(TEXT_BLOCK_OT_cleanup_texts)
    bpy.utils.register_class(TEXT_BLOCK_PT_panel)
    
    # Добавляем обработчик события для смены текстового блока при выборе объекта или коллекции
    bpy.app.handlers.depsgraph_update_post.append(depsgraph_update_post)

def unregister():
    bpy.utils.register_class(TEXT_BLOCK_OT_create_or_open)
    bpy.utils.register_class(TEXT_BLOCK_OT_delete_text)
    bpy.utils.register_class(TEXT_BLOCK_OT_cleanup_texts)
    bpy.utils.register_class(TEXT_BLOCK_PT_panel)
    
    # Удаляем обработчик события
    bpy.app.handlers.depsgraph_update_post.remove(depsgraph_update_post)

if __name__ == "__main__":
    register()
