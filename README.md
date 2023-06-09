This script creates an environment variable for the current scene. It can be used in texture paths.<br>
Created `scriptNode` sets the environment variable when the scene is opened or referenced.

```python
import pymel.core as pm
import os

def n_dirname(path, n=0):
    parent = os.path.dirname(path)
    return parent if n <= 0 else n_dirname(parent, n-1)
    
def createSceneEnvVar(sceneEnvVar, depth=0): # 0-current asset folder, 1-parent folder, etc
    if pm.objExists(sceneEnvVar):
        pm.warning("'{}' already exists".format(sceneEnvVar))
        return
    
    currentFile = pm.api.MFileIO.currentFile()
    if currentFile.endswith("untitled"):
        pm.warning("Save scene first")        
        return
        
    code = '''
import maya.cmds as cmds
import maya.OpenMaya as om

def n_dirname(path, n=0):
    dirname = lambda path: "/".join(path.split("/")[:-1])                  
    parent = dirname(path)
    return parent if n <= 0 else n_dirname(parent, n-1)
    
secretNode = cmds.ls("$VAR", r=True)
if secretNode:
    if cmds.referenceQuery(secretNode, isNodeReferenced=True):         
        refNode = cmds.referenceQuery(secretNode, rfn=True)
        refPath = cmds.referenceQuery(refNode, f=True)
    else:
        refPath = om.MFileIO.currentFile()
               
    path = n_dirname(refPath.replace("\\\\", "/"), $DEPTH)
    os.environ["$VAR"] = path
    print("Set $VAR to '{}'".format(path))'''.replace("$VAR", sceneEnvVar).replace("$DEPTH", str(depth))
       
    pm.scriptNode(st=1, bs=code, n="sceneEnv_scriptNode", stp="python")        
    pm.createNode("transform", n=sceneEnvVar) #  make secret node
    exec(code) # setup env
        
    # change file textures path_script
    sceneRoot = os.path.normpath(n_dirname(pm.api.MFileIO.currentFile(), depth))
    for n in pm.ls(type="file"):
         txPath = os.path.normpath(n.fileTextureName.get())
         if txPath.startswith(sceneRoot):
            txPath = txPath.replace(sceneRoot, "$"+sceneEnvVar)
            n.fileTextureName.set(txPath)
            print("{} => {}".format(n, txPath))     
```

Run the following function to do the job.
```python
createSceneEnvVar("assetName_path", 0) # depth=0-scene's folder, 1-parent folder, etc
```
Where `assetName_path` can be any unique name you like for your character, `depth` specifies how many time it should go up relative to the current scene's folder.<br>
Use `depth=0` when you want the environment variable points to the scene's folder, `depth=1` points to the parent of the scene's folder, etc.<br>
Place created transform node `sceneEnvVar` to any group you like or hide it from outliner.
