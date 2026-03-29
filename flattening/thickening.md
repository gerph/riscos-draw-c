We now want to thicken the lines that were produced by the flattening process.
Thickening means taking the lines that were created and producing wider versions
of them as a path, running parallel to the input lines. The thickening should turn
each line in to a section of lines which have a thickness of THICKNESS, centred on
the input line (that is, THICKNESS/2 on either side of the lines. Because we have
flattened the path already, only Draw_MoveTo and Draw_LineTo elements will be in
the list. Corners of the lines should be extended until the thickened lines
meet.

