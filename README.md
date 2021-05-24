# Small-Shell-for-the-TMS9900
A small Shell programme to front end the disc and io monitor and to complete the DOS.

It is based on James Hendrix 8080 code written as Small VM/Shell to complement North Star DOS and was featured in a 1980s article in Dr Dobb's.

This TMS9900 version uses the existing structure but has been recoded to run on a TMS9900 CPM like system, but 
the Shell incorporates additonal CPM type commands such as DIR, SAVE and ERA. 

The Shell prompts with the % character.

**%DIR**  -> List the DOS directory.

**%ERA** FileName -> Erases a file from the DOS directory.
  
      Example, ERA TEST.TXT will erase the file TEST.TXT from the disc.
  
**%SAVE**  N  FileName -Load Address  -> Saves the contents of memory (normally and executable) at the
  <LOAD ADDRESS> to disc.  N * 512 bytes are written to disc.
  
     Example, %SAVE 6 XMODEM -0100 will save 3k of data located at address 0100H to disc.   


