https://jinxxy.com/SentFromSpaceVR/robust-weight-transfer

Basic cam control (middle mouse, shift+ middle mouse and scroll)
Focus cam (numpad .)

1. Import
    - Import avi base and clothing
    - Hide everything unimportant
2. Setting up the new armature
    - Duplicate new armature (shift+d, enter)
    - Rename old armature and rename new armature
    - Parent clothes to new armature (ctrl + p, object)
    - Set armature modifier on each mesh to the new armature (blue Wrench)
    - Edit armature and delete bones you do not need for clothes (shift+g, children, delete) (delete the bones in the armature you parented the clothes too)
    - Optional: add extra bones from clothes to new armature and position them
3. Fitting the clothes
    - Major edits using edit mode
    - Enable mirror mode on x
    - Use proportional editing (O)
    - moving: g , (x,y,z)
    - scaling: s , (x, y, z)
    - Use x-ray for selecting (alt+z)
    
    - Smaller edits using sculpt mode
    - Elastic grab/grab brush
    - Smooth brush (surface mode for minor fixes)
    - Inflate brush
    - (Maybe pose tool for certain edits)

    - Position custom bones of the clothes to the right area after the clothes are fitted properly
4. Weight paint
    - Delete old vertex groups that need to be redone (keep the ones for custom bones of the clothes!) klick on cloth -> data -> delete all vertex groups of the human body bones ofc. twist bones related to tho.
    - Weight transfer with the SENT plugin
    - Set up view
        - Armature display as stick, always in front (armature -> data -> expand viewport display )
        - Mesh: viewport display as wireframe
    - Set the base body armature modifier to your clothing armature
    - Select armature, ctrl+click the mesh you want to weightpaint and enter weight paint mode (allows you to move bones while weightpainting)
    - Ensure mirror x is on
    - alt+click to select bones
    - rotate: r, (x,y,z)
    - reset bone position, select all (a), reset (alt+r)
    - Editing weight paints with paint brush tool, set weight to 1 or 0 for adding or substracting, set strength low for small edits
    - Blur tool to smooth things out
    - Pose bones to find issues, adjust weights to fix it
    - Sometimes weight painting cant fix it, use sculpting to fix problem areas

5. Export to vr
    - File > export > fbx
    - Limit to selected objects, and select only your clothing and its armature
    - object types, armature and mesh
    - Apply scalings: FBX All
    - Armature: add leaf bones OFF

    - Import in unity, set fbx settings: read write on, legacy blendshape normals on 
    - Do your usual modular or vrcx stuff for armature, toggles whatever

    - Pro tip, use a posing tool like gogoloco or such to test your weight paint job, and hop back in to blender to fix issues

    - To update the fbx, open the fbx in your windows folders and then replace the file with a new export, this will make unity update it without you having to redo everything.