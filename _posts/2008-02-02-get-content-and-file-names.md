---
layout: post
---
Another day, another PowerShell feature discovered. Unfortunately this time it was a feature that made me think I had a bug in my script. The script read through some directories, did some file parsing and created a data object for every directory in the form of a [Tuple]({% post_url 2007-11-29-tuples-in-powershell %}).  

One of the files was People.txt and contained a list of relevant people for the data (one per line). Unfortunately the parameter kept showing up like so
...

    People  
    \------  
    {People.txt, People.txt, People.txt, People.txt}

Confusion set in as to what I did wrong since the code in question amounted to the following.

    D:\temp> new-tuple "People",(gc People.txt)

Next step is manual verification

    D:\temp> gc People.txt  
    jared  
    jamie  
    jason  
    mary

So far so good. Then a tried the explicit tuple code and got back the People.txt problem. Eventually I decided to try out get-member and see exactly what I had in the array. It was a string type as expected but then I noticed the following ... extra items.

    D:\temp> gc People.txt | gm

    '? TypeName: System.String

    Name'''''''' MemberType'''''''? Definition  
    \----'''''''' \----------'''''''? \----------  
    ...

    PSParentPath''?? NoteProperty'''''' System.String PSParentPath=D:\temp  
    PSPath''''''?? NoteProperty'''''' System.String PSPath=D:\temp\People.txt  
    PSProvider'''' NoteProperty
    System.Management.Automation.Provider...  
    ReadCount''''?? NoteProperty'''''' System.Int64 ReadCount=1  

It turns out that when you read information from a file in PowerShell they will handily add in the file information as NoteProperty instances. All well and good and I can see where that would be useful but for now it's really messing up my display. To remove it just tell powershell that I really just want a string.

    D:\temp> new-tuple "People",(gc People.txt | %{[string]$_})

    People  
    \------  
    {jared, jamie, jason, mary}

