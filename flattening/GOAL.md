This project is meant to flatten the bezier curve within the Draw_PathElement structure into a number of distinct lines. The 
input path `inpath` contains the path we want to manage. The output path contains the data we're writing out. In the outpath 
structure, there is a Draw_EndPath element which will indicate the end of the path, and has a parameter (in end_path) which 
contains the amount of space remaining in the buffer. The Draw_EndPath element should be replaced with the new flattened 
path. In the flattened path, the MoveTo and LineTo elements should be copied. The BezierTo elements however should be 
converted from the 2 control points plus an end point into a number of LineTo elements. The FLATTENING parameter is the 
distance between line elements in the flattened path. Implement the `flatten` function. You can build the code with 
`riscos-amu` and run it with the `riscos-build-run aif32 --command aif32.DrawFlattening` command.  Ask questions if you need 
to. The draw.xml document describes the data structures, but you cannot call the SWIs within it.

