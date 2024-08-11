# blender_mixamo_alpha_script
Early alpha script to import and merge multiple mixamo animations into one model as separate NLA tracks

```


#START OF SCRIPT

bl_info = {
    "name": "MIMT",
    "author": "Rhys Thomas",
    "version": (0, 0, 1),
    "blender": (2, 80, 0),
    "location": "3D Viewport > Sidebar > Mixamo Anims",
    "description": "import multiple mixamo animations and instantly merge into an existing armature as separate NLA action tracks",
    "category": "Mixamo Anims",
}

# give Python access to Blender's functionality
import bpy


class VIEW3D_PT_MIMT_panel(bpy.types.Panel):  # class naming convention ‘CATEGORY_PT_name’

    # where to add the panel in the UI
    bl_space_type = "VIEW_3D"  # 3D Viewport area (find list of values here https://docs.blender.org/api/current/bpy_types_enum_items/space_type_items.html#rna-enum-space-type-items)
    bl_region_type = "UI"  # Sidebar region (find list of values here https://docs.blender.org/api/current/bpy_types_enum_items/region_type_items.html#rna-enum-region-type-items)

    bl_category = "MIMT"  # found in the Sidebar
    bl_label = "Mixamo Import Merge Tweak"  # found at the top of the Panel

    def draw(self, context):
        layout = self.layout
        scene = context.scene
        """define the layout of the panel"""
        row = self.layout.row()
        row.operator("mixamo.import_merge_tweak", text="Import FBX Animations")
        row.prop(scene, "in_place")
        


def register():
    bpy.types.Scene.in_place = bpy.props.BoolProperty(
        name="Animate In Place",
        description="each animation is forced to its start frame location on the axis of largest movement, eg if animation moves forward on the Z, it will be forced not too. Animation plays in place!",
        default = True)
    bpy.utils.register_class(VIEW3D_PT_MIMT_panel)

def unregister():
    bpy.utils.unregister_class(VIEW3D_PT_MIMT_panel)
    del bpy.types.Scene.in_place



import bpy
import bpy.ops
from bpy.props import StringProperty, BoolProperty
from bpy.types import Operator
import os # This module provides a unified interface to a number of operating system functions.
import sys # This module provides a number of functions and variables that can be used to manipulate different parts of the Python runtime environment.
import math
#from math import mathutils
class MIMT(Operator):

    """Select Folder To Import .."""
    bl_idname = "mixamo.import_merge_tweak"
    bl_label = "Import, Merge and Tweak Mixamo Anims"
    bl_options = {'REGISTER'}

    # Define this to tell 'fileselect_add' that we want a directoy
    directory: StringProperty(
        name="Import Path",
        description="Where to pull mixamo animations from"
        # subtype='DIR_PATH' is not needed to specify the selection mode.
        # But this will be anyway a directory path.
        )

    # Filters folders
    filter_folder: BoolProperty(
        default=True,
        options={"HIDDEN"}
        )

    
    def execute(self, context):
        originalSpace = bpy.context.space_data;
        originalObjects = bpy.context.view_layer.objects.active;
        originalArea = bpy.context.area.type;
        originalContext = context;
        originalMode = bpy.context.mode;
        print("original c : " + str(originalContext) + "");
        print("original space : " + str(originalSpace) + "");
        print("original objects : " + str(originalObjects) + "");
        
        def ShowMessageBox(message = "", title = "Import and Merge", icon = 'INFO', finish = False):

            def draw(self, context):
                self.layout.label(text=message)

            bpy.context.window_manager.popup_menu(draw, title = title, icon = icon)
            if finish == True:
                unregister()
            else:
                print("not unregistering after message")
                
            return {'FINISHED'}
        
        def DebugContexts():
            debugSpace = bpy.context.space_data;
            debugObjects = bpy.context.view_layer.objects.active;
            debugArea = bpy.context.area.type;
            debugContext = context;
            debugMode = bpy.context.mode;
            debugActiveObject = bpy.context.active_object;
            commonDebugObjects = [['space', debugSpace], ['objects', debugObjects], ['area', debugArea], ['context', debugContext], ['mode', debugMode], ['active_object', debugActiveObject]]
            for i in commonDebugObjects:
                try:
                    PrintObjectDebug(i[1], i[0]);    
                except IndexError:
                    print(' IndexError ');
                except KeyError:
                    print(' KeyError ');
                except LookupError:
                    print(' LookupError ');
                except AssertionError:
                    print(' AssertionError ');
                except:
                    print('Some other error : ' + str(i[0]) + ' ~~~~~ ' + str(i[1]));
                finally:
                    print('\n\n')
            return {'FINISHED'}
        
        def PrintObjectDebug(dobj, label):
            if dobj != None:
                print("\n\n Debugging info for : " + str(label) + "\n\n   -> " + str(dobj));
                properties = [['name', dobj.name],['type', dobj.type], ['data', dobj.data], ['active', dobj.active], ['action', dobj.action], ['object', dobj.active_object], ['mode', dobj.mode]]
                for i in properties:
                    try:
                        print('Obj : [ ' + str(dobj) + ' | ' + label + ' has property ' + i[0] + ' -> ' + str(i[1]) + ' ] \n ');
                    except:
                        print('Debug of ' + label + 'failure');
            return {'FINISHED'}
        
        def SelectDest(obj):
            
            currentCon = bpy.context;
            currentArea = bpy.context.area.type;
            currentMode = bpy.context.mode;
            
            if bpy.context.active_object != None:
                print("current active object..." + str(context.active_object.name));
                bpy.ops.object.mode_set(mode='OBJECT');
            else:
                print("no active object");
            if bpy.ops.object != None:
                print("operational object..." + str(bpy.ops.object.name));
                #bpy.ops.object.mode_set(mode='OBJECT');
            else:
                print("no operational object");
            bpy.context.area.type = 'VIEW_3D';
            print('test current context =' + str(currentCon) + ' - ' + str(originalContext) + ' - ' + str(originalSpace) + ' - ' + str(originalObjects) + ' - ' + str(originalArea) + ' - ' + str(originalMode))
            print('selecting' + str(obj.name) + '');
            bpy.ops.object.select_all(action='DESELECT') # Deselect all objects
            bpy.context.view_layer.objects.active = obj   # Make the cube the active object 
            obj.select_set(True);
            bpy.context.area.type = currentArea;
            bpy.ops.object.mode_set(mode=currentMode);
            
            return {'FINISHED'}
        
        print("Selected dir -> '" + self.directory + "'")
        
        path = self.directory
        dir = os.listdir(self.directory)
        active_object = None;
        destArm = None;
        if len(bpy.context.selected_objects) == 1:
            print("Currently selected : " + bpy.context.view_layer.objects.active.name + "");
            active_object = context.active_object;
            selected_armature=(active_object is not None and active_object.type == 'ARMATURE');
            print("correct object type selected :" + str(selected_armature) + "");
            if selected_armature == True:
                destArm = active_object;
            else:
                ShowMessageBox("selected object is not type: ARMATURE, currently :" + str(active_object.type) + "", "Selection ERROR", "ERROR", True);
        else:
            print("No armature selected");
            ShowMessageBox("No armature selected, will attempt to use object named 'Armature'", "Automate Info", "INFO");
            destArm = bpy.context.scene.objects.get("Armature", None)
        
        
        if destArm == None:
            ShowMessageBox("No armature selected OR no armature in scene with default name", "Armature ERROR", "ERROR", True);
                    
        else:
            SelectDest(destArm);
            for file_path in dir:
                    if file_path.lower().endswith('.fbx'):
                            print ("individual files : " + str(path + file_path) + "")
                            fullpath = str(path + file_path);
                            print( "scene objects - " + str(bpy.context.scene.objects) + "" );
                            print("check");
                            bpy.ops.import_scene.fbx(filepath=fullpath);
                            newname = file_path[:-4]
                            impObj = context.active_object;
                            impObj.name = newname;
                            objData = impObj.data;
                            objData.name = newname;
                            animData = impObj.animation_data;
                            action = animData.action;
                            action.name = newname;
                            print("check");
                            SelectDest(destArm);
                            bpy.ops.object.posemode_toggle();
                            bpy.ops.object.mode_set(mode='POSE');
                            currentArea = bpy.context.area.type;
                            currentSpace = bpy.context.space_data;
                            bpy.context.area.type = 'DOPESHEET_EDITOR'
                            bpy.context.space_data.mode = 'ACTION'
                            bpy.ops.action.new();
                            bpy.context.area.type = currentArea;
                            bpy.ops.object.mode_set(mode='OBJECT');
                            destArm.animation_data.action = action;
                            bpy.ops.object.mode_set(mode='POSE');
                            currentArea = bpy.context.area.type;
                            bpy.context.area.type = 'NLA_EDITOR'
                            bpy.ops.nla.action_pushdown(track_index=1);
                            bpy.context.area.type = originalArea;
                            bpy.context.view_layer.objects.active = None;
                            deleteObj = bpy.context.scene.objects[newname];
                            bpy.ops.object.select_all(action='DESELECT') # Deselect all objects
                            bpy.context.view_layer.objects.active = deleteObj   # Make the cube the active object 
                            bpy.context.view_layer.objects.active.select_set(True);
                            print("all objects " + str(bpy.context.scene.objects) + "" );
                            bpy.ops.object.delete();
                            print("imported Object deleted ?? " + str(bpy.context.scene.objects) + "" );

                        
        
            bpy.context.area.type = 'VIEW_3D';
            SelectDest(destArm);
            bpy.ops.object.mode_set(mode='POSE');
            bpy.context.area.type = 'NLA_EDITOR';
            print("check this runs");
            print("check contexts = " + str(originalArea) + " - " + str(originalContext) + " - " + str(originalMode) + "");
            if bpy.context.pose_object != None:
                animationData = bpy.context.pose_object.animation_data;
                alltracks = animationData.nla_tracks;
                
                for track in alltracks:
                    alltracks.active = None;
                    track.select = True;
                    alltracks.active = track;
                    if bpy.context.active_nla_track != None:
                        for strip in track.strips:
                            strip.select = True;
                            insideAction = strip.action;
                            destArm.animation_data.action = insideAction;
                            bpy.ops.anim.channels_move(direction='TOP');
                            print("context active action = " + str(bpy.context.active_action));
                        bpy.data.scenes['Scene'].frame_set(1);
                        bpy.data.scenes['Scene'].frame_set(2);
                        bpy.data.scenes['Scene'].frame_set(1);
                        bpy.data.scenes['Scene'].show_keys_from_selected_only = True;
                        print("do some stuff" + alltracks.active.name + " | " + bpy.context.active_nla_track.name + "")
                        hipbone = bpy.context.visible_pose_bones[0];
                        print('bone name ->>>' + hipbone.name);
                        locframe1 = hipbone.location.copy();
                        bpy.data.scenes['Scene'].frame_set(99999);
                        bpy.data.scenes['Scene'].frame_set(1);
                        bpy.data.scenes['Scene'].frame_set(99999);
                        locframe2 = hipbone.location.copy();
                        xd = math.fabs(locframe1[0] - locframe2[0]);
                        yd = math.fabs(locframe1[1] - locframe2[1]);
                        zd = math.fabs(locframe1[2] - locframe2[2]);
                        largest = None;
                        if xd > yd and xd > zd:
                            largest = 0;
                            
                        if yd > xd and yd > zd:
                            largest = 1;
                            
                        if zd > yd and zd > xd:
                            largest = 2;
                            
                        if largest != None:
                            copyValue = float(locframe1[largest])+1;
                            replaceValue = float(copyValue)-1;
                            rf = insideAction.frame_range;
                            ef = int(rf[1]);
                            sf = int(rf[0]);
                            print("action - " + insideAction.name + " range = " + str(rf));
                            for i in range(ef):
                                bpy.data.scenes['Scene'].frame_set(i+1);
                                copyBoneLocationFrame = hipbone.location.copy();
                                copyBoneLocationFrame[largest] = replaceValue;
                                if context.active_object != None:
                                    bpy.ops.object.mode_set(mode='POSE');
                                    bpy.context.area.type = 'NLA_EDITOR';
                                    #DebugContexts();
                                    try:
                                        hipbone.keyframe_delete("location", frame=i+1)
                                        hipbone.location = copyBoneLocationFrame;
                                        hipbone.keyframe_insert("location", frame=i+1)
                                    except:
                                        print('action : ' + insideAction.name + ' caused error whilst attempting to fix bone position');
                                
                        print("get difference > " + str(locframe1) + " ? " + str(locframe2) + " = ");
                        print("largest diff = " + str(largest));
                    bpy.ops.anim.channels_move(direction='BOTTOM');
                    strip.select = False;
                    track.select = False;
                    alltracks.active = None;
                    
                return {'FINISHED'}
                
        return {'FINISHED'}
        unregister();

    def invoke(self, context, event):
        # Open browser, take reference to 'self' read the path to selected
        # file, put path in predetermined self fields.
        # See: https://docs.blender.org/api/current/bpy.types.WindowManager.html#bpy.types.WindowManager.fileselect_add
        context.window_manager.fileselect_add(self);
        return {'RUNNING_MODAL'}


def registerMIMT():
    bpy.utils.register_class(MIMT)


def unregisterMIMT():
    bpy.utils.unregister_class(MIMT)
 

#
# Invoke register if started from editor
if __name__ == "__main__":
    register()
    registerMIMT()
    
    # OPTIONAL - test run
    #bpy.ops.mixamo.in_place_fix('INVOKE_DEFAULT')
    

#END OF SCRIPT



```

