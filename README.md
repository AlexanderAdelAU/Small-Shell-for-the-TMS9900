# Small-Shell-for-the-TMS9900
A small Shell programme is a Command Processor for the DOS.  Its purpose is to provide the user interface to allow interaction with the DOS.

It is based on James Hendrix 8080 code written as Small VM/Shell to complement North Star DOS and was featured in a 1982 article in Volume 7 of Dr Dobb's Journal (Vol 7 Number 63).  The article provides a full description of the procedural commands that are available. https://ia600109.us.archive.org/17/items/dr_dobbs_journal_vol_07_201803/dr_dobbs_journal_vol_07.pdf

This TMS9900 version uses the existing structure but has been recoded to run on a TMS9900 CPM like system, but 
the Shell incorporates additonal CPM type commands such as DIR, SAVE and ERA. 

The Shell prompts with the % character.

**%DIR**  -> List the DOS directory.

**%ERA** FileName -> Erases a file from the DOS directory.
  
      Example, ERA TEST.TXT will erase the file TEST.TXT from the disc.
  
**%SAVE**  N  FileName -Load Address  -> Saves the contents of memory (normally and executable) at the
  <LOAD ADDRESS> to disc.  N * 512 bytes are written to disc.

  
     Example, %SAVE 6 XMODEM -0100 will save 3k of data located at address 0100H to disc.   
  
    
You can then execute the programme by simply typing the name of the saved programme, in this case:
  **%XMODEM -> Executes the programme.

and **%.XMODEM**  -> Just loads the programme into memory.   Placing a full stop in front of the filename loads file into memory and returns to Shell.  This is useful if you wish to inspect the disc file or patch it.
