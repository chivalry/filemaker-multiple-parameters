This was never an original article, but a comment I made to [Matt Petrowski's][Matt Petrowski] screencast about [function scripting in FileMaker].

I won't cover the details (see the [FileMaker help] for that), but basically the problem being solved here is that while FileMaker scripts can accept a parameter, it's only a single parameter, and often it would be very useful to pass multiple parameters to a script. An example I often use would be sending a record type and record id to a script that deletes a record.

Matt originally found the technique from Alexander Zueiv and Mikhail Edoshin, but the original articles he links to are both gone, but if I find them I'll edit this to link to them. Matt took their technique directly, I think, and I edited it in a number of ways. First of all, the original `AssignParams` function was quite large, so I broke it into multiple custom functions. Second, I changed the template of the function names to use parentheses to enclose all the parameters and curly brackets to enclose optional parameters, as that's more similar to the syntax of most programming languages (the original reversed the use of parenthese and curly brackets). However, the way it's written the delimiters are stored as variables and can therefore be easily changed.

You can [download the technique file directly] or check out the [github repository] for any updates.

There are other ideas on how to handle this. [Jesse Antunes][] at [Six Fried Rice][] has an [article][Six Fried Rice Article] that uses key/value pair syntax to indicate multiple parameters, and the [FileMaker Dictionary Functions] offered by Six Fried Rice have been used by [Thomas Seidler] to [solve the problem]. I like the following solution, however, because I find it simpler (believe it or not) and I enjoy the self documenting script names.

# The Goal
The basic idea behind this technique is to not only allow FileMaker scripts to accept multiple parameters, but ensure that the parameters are what the script expects and automatically assign the parameters to variables. So if we begin, for example, with a script named `Delete Record( RecordType; RecordID )`, the end result is that we ensure both those parameters are passed and when they are, their values are assigned to variables, `$RecordType` and `$RecordID`.

After all of these functions have been added to a database's custom funcitons, when a script needs to accept parameters, the name of the script will indicate what parameters are expected, which are required and which are optional. In the above example, `Delete Record( RecordType; RecordID )`, the script expects two parameters and both are required. As an example of a script with optional parameters, `New Recod( RecordType {; ParentRecordID} )` expects at most two parameters, the `RecordType` is required and `ParentRecordID` is optional.

So if a call is made to `Delete Record( RecordType; RecordID )` without both of the parameters, an error can be generated. Similarly, if a call is made to `New Record( RecordType {; ParentRecordID} )` and the first parameter is not sent, an error can be generated, but if the second parameter is unsnet, the script won't generate an error. If it's included, however, a variable with that name will be assigned within the script.

The system includes one more way to specify parameters, those that are mutually exclusive. `Go to Tab( [Next|Previous] )` is interpreted to mean that either the `Next` parameter is sent or the `Previous` parameter is sent, but not both. In this case, you would send `True` to one and only one of them.

When writing the script, you use the `AssignParams` function to both check that the parameters are valid and to assign the valid parameters to local variables.

    If[AssignParams]
      // Do the script stuff here.
    Else
      // Generate an error here.
    End If

# Utility Functions
First I need to cover a couple of utility functions. These are standard functions in my custom function library, and I find them useful apart from this particular technique. Some of these you can ommit if you don't like the idea of simple convenience functions.

## `Null`
I use this as a more human-readable way to specify an empty string, especially since I'm often setting something to an empty string in order to remove the script variable from memory ( or at least the list of variable shown in FileMaker's Data Viewer) or because I'm using a `Let` function to perform some task but don't need to do anything with the actual results of that function.

    ""

## `LeftWord( _words )`
Returns the first word from the string of words passed.

	// Returns only the leftmost word of the passed text.	//	// Written by Charles Ross		LeftWords( _words; 1 )

## `FirstListItem( _list )`
Simply takes a return-separated list of items and returns the first item in the list.

    // Returns the first value in a list, including the ending paragraph mark. Often    //   used to loop through a list of items. Returns the item without the trailing    //   carriage return.    //    // Written by Charles Ross    GetValue( _list; 1 )## `RestOfList( _list )`Kind of the opposite of `FirstListItem`, in that it returns a new list that excludes the first item. These two functions together allow the easy iteration through a list of tiems in a loop or recursive function.
    // Returns the list without the first item. Usually used to loop through a list    //   of items.    //    // Written by Charles Ross        RightValues( _list; ValueCount( _list ) - 1 )
## `WordsToList( _words )`Converts the words in the string passed into a list of words.
	// Converts the text passed into a list of words. For example, WordsToList(	//   "one two>three;four" ) = "one¶two¶three¶four¶".	//	// Written by Charles Ross		Case(	  WordCount( _words ) = 0;	  "";	  LeftWord( _words ) & "¶" &	    WordsToList( RightWords( _words; WordCount( _words ) - 1 ) )	)
# Technique FunctionsHere are the functions in the order they should be added to your custom functions (all the utility functions should already be in place). Hopefully the comments within the functions are sufficient to guide understanding (obviously, let me know if you have questions), so I'll provide a simple example that uses a script and all of the possible parameter types (required, optional and mutually exclusive) and walk through what the functions do and what the final result it, both in terms of what's returned by `AssignParams` and what script variables are assigned from the parameters.## `SingleParamToVar( _paramName )`
	// Assigns the single parameter named to a local script variable. $Params is a	//   variable declared in the custom function SetAssignParamVars. It's value is	//   simply Get( ScriptParameter ). The _paramName may not have been passed (it	//   may be optional to the script), so we check for its existence before doing	//   anything. The function doesn't actually return anything useful, but after	//   it's run, the variable named in _paramName should have a local script	//   variable declared with that name if the caller of the script passed such a	//   parameter.	//	// EXTERNAL REQUIREMENTS: The SetAssignParamVars custom function (to declare the	//   $Params script variable).	//	// Written by Charles Ross. Inspired by Alexander Zueiv.		Case(	  // Does the parameter exist in the arguments to the script?	  PatternCount( $Params; _paramName );  	  Let( [	      // Get the value of the named parameter by evaluating within a Let	      //   function the passed parameter and returning in that statement the	      //   named parameter.	      Value = Evaluate ( "Let ( [ " & $Params & "] ; " & _paramName & " )" );		      // Construct a local script variable and assign it the value found above.	      x = Evaluate( "Let( [ $" & _paramName & " = \"" & Value & "\" ]; \"\" )" )	    ];		    // Return the value of the local script variable. This is a debugging	    //   feature to make sure it works correctly.	    Evaluate( "$" & _paramName )	  );		  "" // Empty string returned if the parameter doesn't exist.	)
## `SetAssignParamVars`
	// Store constants and other variables needed during script parameter	//   validation.	//	// EXTERNAL REQUIREMENTS: The WordsToList custom function.	//	// Written by Charles Ross. Inspired by Alexander Zueiv.		Let ( [	  $Script = Get( ScriptName );	  $RawParams = Get( ScriptParameter );	  $Params = Case(	    Right( $RawParams; 2 ) = "; ";	    Left( $RawParams; Length( $RawParams ) - 2 );	    $RawParams	  );		  $OpenChar     = "(" ;    // beginning of parameters definition	  $CloseChar    = ")";     // end of parameters definition	  $BreakChar    = ";" ;    // regular parameters separator (and)	  $AltOpenChar  = "[";     // beginning of alternative parameters definition.	  $AltCloseChar = "]";     // end of alternative parameters definition.	  $AltChar      = "|" ;    // alternative parameters separator (or)	  $OptionalChar = "{" ;    // beginning of optional parameters section		  StartPos = Position( $Script; $OpenChar; 1; 1 ) + 1;	  ParamLen = Position( $Script; $CloseChar; Length( $Script ); -1 )	    - Position( $Script; $OpenChar; 1; 1 ) - 1;	  $ParamTemplate = Middle( $Script; StartPos; ParamLen );		  // Convert the parameters to a list that can be interated through.	  $ParamList = WordsToList( $ParamTemplate );		  $EmptyParamTemplate = "( not IsEmpty( $ ) )"	]; "" )

## `ParamToVars( _paramList )`
	// A recursive function that declares as local script variables each of the	//   named varaibles in ParamList. ParamList is calculated from the name of the	//   script.	//	// EXTERNAL REQUIREMENTS: The FirstListItem and RestOfList custom functions	//   (list operations) and the SingleParamToVar custom function.	//	// Written by Charles Ross. Inspired by Alexander Zueiv.		Case(	  ValueCount( _paramList ) = 0;	  "";	  SingleParamToVar( FirstListItem( _paramList ) ) &	    ParamToVars( RestOfList( _paramList ) )	)
## `AssignParams`
	// A rather complicated function that not only assigns parameters to local	//   script variables, but also validates them given the template provided by	//   the called script's name. Returns True if required parameters are passed	//   and at least one of the alternative parameters are included. Returns False	//   otherwise. But in the process of determining the return result, also	//   assigns the parameters to local script variables.	//	// EXTERNAL REQUIREMENTS: A number of custom functions that perform many sub	//   parts, such as actually assigning the local script variables and setting up	//   the local variables needed to operate: SetAssignParamVars, WordsToList,	//   ParamToVars, SingleParamToVar.	//	// Written by Charles Ross. Inspired by Alexander Zueiv.		Let ( [	  //----------------------------------------------------------------------------	  x = SetAssignParamVars;         // Set up the local script variables used in	                                  //   this custom function set.		  //----------------------------------------------------------------------------	  x = ParamToVars( $ParamList );  // Convert the script parameter passed and	                                  //   $Parsed into local script variables.		  //----------------------------------------------------------------------------	  $Troubleshoot = False;          // If troubleshoot is false, clear out the	                                  // script variables when finished processing.	                                  	  //----------------------------------------------------------------------------	  $Parsable = Substitute(         // Ease parsing by removing spaces. We'll add	                                  //   the dollar sign ourselves.	    $ParamTemplate;	    [ " "  ; "" ];	    [ "$" ; "" ]	  );		  //----------------------------------------------------------------------------	  $ReqParams = Case(              // Remove optional parameters.	    PatternCount( $Parsable; $OptionalChar );	    Left( $Parsable; Position( $Parsable; $OptionalChar; 1; 1 ) - 1 );	    $Parsable	  );		  //----------------------------------------------------------------------------	  $Parsed = "$" & Substitute(     // Enclose alternate possibilities in	                                  //   parentheses and prepend parameter names	                                  //   with the dollar sign.	    $ReqParams;	    [ $BreakChar & $AltOpenChar ;   ";( $"                       ];	                                  // Enclose the optional parameters in	                                  //   parentheses.	    [ $AltChar                               ;   $AltChar & "$"  ];	                                  // Add dollar signs before each optional	                                  //   parameter after the first.	    [ $AltCloseChar                       ;   " )"               ]	                                  // Close the optional parameters parentheses.	  );		  //----------------------------------------------------------------------------	  $Parsed = Substitute(           // Handle the possible special case with the	                                  //   break character is followed by	                                  //   alternative parameters.	    $Parsed;	    [ $BreakChar & "("; "^^^^"                 ];	                                  // Substitute an unlikely string for the break	                                  //   sequence we want to keep.	    [ $BreakChar        ; $BreakChar & "$" ];	    [ "^^^^"               ; $BreakChar & "("  ] );	                                  // Substitute the break sequence we want to	                                  //   keep back in.		  //----------------------------------------------------------------------------	  // Add in the FileMaker code to ensure that the appropriate strings are not	  //   empty, thus validating that each parameter that is required is present.	  $FMCode = "( not IsEmpty( " & Substitute(	    $Parsed;	    [ $BreakChar; " ) ) and ( not IsEmpty( " ];	    [ $AltChar; " & " ]	  ) & " ) )";	  	  //----------------------------------------------------------------------------	  // Store the result before we possibly clear the variables.	  _result = ( $FMCode = $EmptyParamTemplate ) or Evaluate( $FMCode );	  	  //----------------------------------------------------------------------------	  // Clear out the script variables if we're not troubleshooting the function.  	  x = Case(	    not $Troubleshoot;	    Let(	      [	        $Troubleshoot       = Null;	        $Parsable           = Null;	        $ReqParams          = Null;	        $Parsed             = Null;	        $Script             = Null;	        $RawParams          = Null;	        $Params             = Null;	        $OpenChar           = Null;	        $CloseChar          = Null;	        $BreakChar          = Null;	        $AltOpenChar        = Null;	        $AltCloseChar       = Null;	        $AltChar            = Null;	        $OptionalChar       = Null;	        $ParamTemplate      = Null;	        $ParamList          = Null;	        $FMCode             = Null;	        $EmptyParamTemplate = Null 	     ];	      	     Null	    )	  )		  ];		  _result	)## `Param( _varName; _paramValue )`
	// Encapsulates the building of parameter/value pairs used by the AssignParams	//   function.	//	// Written by Charles Ross.		_varName & " = " & Quote( _paramValue ) & "; "
# Technique Details## `Param`
Let's start with the simplest of the functions, `Param`, which simple provides an easy way to pass parameters to a script by using a call to `Param` for each parameter you want to pass. In the example of the `Delete Record( RecordType; RecordID )` script, calling this script with the appropriate parameters would set the script parameter to something like this:

    Param( "RecordType"; "Invoice" ) &
    Param( "RecordID"; 1423 )

The order of parameters doesn't matter, and we could just as easily have passed the `RecordID` parameter first. All that matters is that each parameter is sent with the appropriate name and value. The important item to note here is that it's very easy to read this and understand what is being passed to the script.

## Foundations
There are a few basic ideas used in this technique. First of all heavy use of script variables is used. We're setting many script varialbes that are set in one function (`SetAssignParamVars`) and used by other functions. We could conceivably avoid this if we used fewer functions, or even a single function, but I find that breaking the feature into multiple functions eases understanding and, while writing it, eased troubleshooting.

Another foundation idea is the use of the [`Evaluate`][] function to execute calculated FileMaker code. The primary use of this is to create a [`Let`][] calculation that will assign a variable name to the required value.

The final foundation also deals with the `Let` function. While it appears that the original purpose of the `Let` function was to simply assign calculation variables within a single calculation, it has the added ability that it can also assign script variables if the variable being assigned is prepended with a dollar sign (or a global variable, if there are two dollar signs prepending the variable name). In other words, the follwoing calculation, while it evaluates to an empty string, has the side effect of also assigning `"abc"` to the `$ScrptVar` variable. It has the exact same effect as the `Set Variable[ $ScriptVar; "abc" ]` script step would have.

    Let(
      [
        $ScriptVar = "abc"
      ];
      
      ""
    )

While you probably would use `Set Variable` if you were assigning a single variable, this feature is quite useful for assigning multiple variables or, as we'll do with this technique, using a script to assign dynamic variables who's names we don't know in advance.

## Example
Let's work with an example script and example parameter string and take a look at what happens at each step. For our script we'll use `Sample Script( Req1; Req2; [ Excl1 | Excl2 ] {; Opt1 } )`. The name of this script tells us that there are five possible parameters, `Req1` and `Req2` are required, `Excl1` and `Excl2` are mutually exclusive (one and only one should be sent), and `Opt1` is optional (can be, but need not be, sent).

Our sample parameter string will be `Param( "Req1"; "Value1" ) & Param( "Req2"; Value2 ) & Param( "Excl2"; True ) & Param( "Opt1"; "OptionalValue" )`.

Assume that we've called our sample script with the sample parameter and are now evaluating the `AssignParams` function within an `If` script step.

The first thing `AssignParams` does is call `SetAssignParamVars`. This simply sets a bunch of script variables, some constant, some calculated. Covering each one (except the constants such as `$OpenChar`, `$CloseChar`, etc.):

    $Script = Get( ScriptName )
            = "Sample Script( Req1; Req2; [ Excl1 | Excl2 ] {; Opt1 } )"
            
    $RawParams = Get( ScriptParameter )
               = Param( "Req1"; "Value1" ) & Param( "Req2"; Value2 ) &
                 Param( "Excl2"; True ) & Param( "Opt1"; "OptionalValue" )                
               = "Req1 = \"Value1\"; Req2 = \"Value2\"; Excl2 = \"1\"; Opt1 = \"OptionalValue\"; "
               
    $Params = Case(
                Right( $RawParams; 2 ) = "; ";
                Left( $RawParams; Length( $RawParams ) - 2 );
                $RawParams
              )
            = "Req1 = \"Value1\"; Req2 = \"Value2\"; Excl2 = \"1\"; Opt1 = \"OptionalValue\""
    
    StartPos = Position( $Script; $OpenChar; 1; 1 ) + 1
             = 15
    
    ParamLen = Position( $Script; $CloseChar; Length( $Script ); -1 ) -
    		   Position( $Script; $OpenChar; 1; 1 ) - 1
    		 = 41
    
    $ParamTemplate = Middle( $Script; StartPos; ParamLen )
                   = " Req1; Req2; [ Excl1 | Excl2 ] {; Opt1 } "
    
    $ParamList = WordsToList( $ParamTemplate )
               = "Req1¶Req2¶Excl1¶Excl2¶Opt1¶"

OK, we have set a bunch of script variables with the use of a couple calculation variables (`StartPos` and `ParamLen`) that we don't need to keep. Now that these script variables have been set they're available to any other functions until the script ends.

Note here the format of the `$Params` variable. If you're already familiar with the `Let` function, you might recognize this as the format for assigning values to variables, which is exactly what we'll eventually use it for.

The next line of `AssignParams` calls `ParamToVars` passing it the `$ParamList` variable. `ParamToVars` is a recursive function that knocks out each available parameter one by one until there aren't any left, assigning each parameter to the named variable. Our exit condition for the recursive function is when there are no more parameters left in the `_paramList` parameter.

Knowing that `$ParamList` at that point has (in this example) a value of `"Req1¶Req2¶Excl1¶Excl2¶Opt1¶"`, let's work through that function.

On the first iteration the function evaluates as follows:

    Case(
      ValueCount( "Req1¶Req2¶Excl1¶Excl2¶Opt1¶" ) = 0;
      "";
      SingleParamToVar( FirstListItem( "Req1¶Req2¶Excl1¶Excl2¶Opt1¶" ) ) &
        ParamToVars( RestOfList( "Req1¶Req2¶Excl1¶Excl2¶Opt1¶" ) )
    )
    
which simplifies to:

    SingleParamToVar( "Req1" ) & ParamToVars( "Req2¶Excl1¶Excl2¶Opt1¶" )

Going through all the recursion and simplifying, we eventually get the equivalent of this:

    SingleParamToVar( "Req1" ) & SingleParamToVar( "Req2" ) & SingleParamToVar( "Excl1" ) & 
    	SingleParamToVar( "Excl2" ) & SingleParamToVar( "Opt1" )

So let's take a look at `SingleParamToVar`, which is the meat of the technique. We won't go through all five iterations, as one example should suffice.

The first call to `SingleParamToVar` passes `"Req1"` as the parameter name. Filling in the variables and simplifying produces (with the application of some more spacing to ease readibility):

	Case(	  PatternCount(
	    "Req1 = \"Value1\"; Req2 = \"Value2\"; Excl2 = \"1\"; Opt1 = \"OptionalValue\"";
	    "Req1" );  	  Let( [	    Value = Evaluate(
          "Let ( [ " & "Req1 = \"Value1\"; Req2 = \"Value2\"; Excl2 = \"1\"; Opt1 = \"OptionalValue\""
          & "] ; " & "Req1" & " )" );	    x = Evaluate( "Let( [ $" & "Req1" & " = \"" & Value & "\" ]; \"\" )" ) ];	    Evaluate( "$" & "Req1" )	  );	  ""	)
    = Let( [	    Value = Evaluate( "Let ( [ Req1 = \"Value1\"; Req2 = \"Value2\"; Excl2 = \"1\"; Opt1 = \"OptionalValue\"" & "] ; Req1 )" );	    x = Evaluate( "Let( [ $" & "Req1" & " = \"" & Value & "\" ]; \"\" )" ) ];	    Evaluate( "$" & "Req1" )	  )

Let's take a closer look at what the calculation variable `Value` is evaluating to. The `Let` function is somewhat hard to read in the format we get with simple substitution, so let's see what it might look like if we were writing the calculation in FileMaker itself.

	Let(
	  [
	    Req1 = "Value1";
	    Req2 = "Value2";
	    Excl2 = "1";
	    Opt1 = "OptionalValue"
	  ];
	
	  Req1
	)

The above is exactly the same as the calculation being passed to `Evaluate` for the `Value` variable, and looking at it this way, you can see that `Value` is going to get a value of `"Value1"`. So our simplication moves to:

	Let( [
	  Value = "Value1";
	  x = Evaluate( "Let( [ $" & "Req1" & " = \"" & Value & "\" ]; \"\" )" )
	  ];
	  Evaluate( "$" & "Req1" )
	)

So what are we doing with that `x` variable? Well, like before, naming the calculation variable `x` indicates we don't care about the return value it's being assigned, only with the what the caluclation itself does. We're evaluating the following string:

    "Let( [ $" & "Req1" & " = \"" & Value & "\" ]; \"\" )"
      = "Let( [ $Req1 = \"Value1\" ]; \"\" )"

Again, formatting that string as if we were entering it directly into FileMaker, it might look like this:

    Let(
      [
        $Req1 = "Value1"
      ];
      
      ""
    )

In this format it's pretty obviously what's happening. We're assigning a value to the `$Req1` script variable while the `Let` function itself ignores the assignment and simply returns an empty string. So after the calculation variable `x` has been set, it's value is simply an empty string, but the side effect of the evaluation is the setting of the script variable `$Req1` to the string `"Value"`.

Our variable is assigned, but let's finish evaluating this function. After all of this simplificaiton, we come to this:

    Let(
      [
        Value = "Value";
        x = ""
      ];
      
      Evaluate( "$Req" )
    )

Since `$Req` now has a value of `"Value1"`, this simply evalutates to that string, `"Value1"`. As the comments say, this value being returned is only there for when debugging the technique, and is otherwise ignored.

So, after all five calls to `SingleParamToVar` we have attempted to assign five script variables, but only four were assigned. `"Excl1"` wasn't because it wasn't passed as a paratmer. In the case of that mutually exclusive parameter, the `PatternCount` test in `SingleParamToVar` fails, so none of the complex portion of its calculation is evaluated, and it simply returns an empty string.

	Case(
	  PatternCount(
	    "Req1 = \"Value1\"; Req2 = \"Value2\"; Excl2 = \"1\"; Opt1 = \"OptionalValue\"";
	    "Excl1" );  
	  Let( … );
	  ""
	)
	= Case(
	    False;
	    Let( … );
	    ""
	  )
	= ""

Remember, we got to this point because `AssignParams` called `ParamToVars`, which in turn recursively called `SingleParamToVar`. So already after the second calculation variable is set in `AssignParams` our variables are already set. The rest of the function deals with using the function template to ensure that required parameters are present.

Toward that end, the next task accomplished by `AssignParams` is to create a `$Parsable` variable by taking the `$ParamTemplate` and removing the spaces and any dollar signs.

    $Parsable = Substitute(
      $ParamTemplate;
      [ " "; "" ];
      [ "$"; "" ]
    )
    = Substitute(
      " Req1; Req2; [ Excl1 | Excl2 ] {; Opt1 } ";
      [ " "; "" ];
      [ "$"; "" ]
    )
    = "Req1;Req2;[Excl1|Excl2]{;Opt1}"

Next we find the required parameters by removing the optional parameters from `$Parsable`.

	$ReqParams = Case(
  	  PatternCount( $Parsable; $OptionalChar );
	  Left( $Parsable; Position( $Parsable; $OptionalChar; 1; 1 ) - 1 );
	  $Parsable
	)
	= Case(
	    PatternCount( "Req1;Req2;[Excl1|Excl2]{;Opt1}"; "{" );
	    Left( "Req1;Req2;[Excl1|Excl2]{;Opt1}"; Position( "Req1;Req2;[Excl1|Excl2]{;Opt1}"; "{"; 1; 1 ) - 1 );
	    "Req1;Req2;[Excl1|Excl2]{;Opt1}"
	  )
	= Left( "Req1;Req2;[Excl1|Excl2]{;Opt1}"; 23 )
	= "Req1;Req2;[Excl1|Excl2]"

Now we go through a few steps to parse what we have and make sure the passed parameters conform to the script template. The first step is to remove the optional character from the required parameters, replacing it with open and close parentheses, and prepend the optional parameters with the dollar sign.

	$Parsed = "$" & Substitute(
	  "Req1;Req2;[Excl1|Excl2]";
  	  [ ";["; ";( $" ];
	  [ "|"; "|$"  ];
	  [ "]"; " )" ]
	)
	= "$Req1;Req2;( $Excl1|$Excl2 )"

Next, prepend the dollar sign to any middle required parameters. We first replace the break character (`;`) and open paraenthsis with some giberish characters, then replace the break character with itself followed by a dollar sign, and then remove the giberish with by replacing it with the break character and the open parenthesis again. We do this so that we're not adding a dollar sign before any open parentheses.

	$Parsed = Substitute(
      $Parsed;
      [ $BreakChar & "("; "^^^^" ];
      [ $BreakChar; $BreakChar & "$" ];
      [ "^^^^"; $BreakChar & "("  ]
    )
    = Substitute(
	  "$Req1;Req2;( $Excl1|$Excl2 )";
	  [ ";("; "^^^^"                 ];
	  [ ";"; ";$" ];
	  [ "^^^^"; ";("  ]
	)
	= "$Req1;$Req2;( $Excl1|$Excl2 )"
	
Now we create some FileMaker calculation code to evaluate. We're going to create a FileMaker calculation that will evaluate to `True` only if all of the required parameters have values. We do this by enclosing each required parameter in a `not IsEmpty` clause and combining the mutually exclusive parameters with concatenation.

	$FMCode = "( not IsEmpty( " & Substitute(
	  $Parsed;
	  [ $BreakChar; " ) ) and ( not IsEmpty( " ];
	  [ $AltChar; " & " ]
	) & " ) )"
	= "( not IsEmpty( " & Substitute(
	  "$Req1;$Req2;( $Excl1|$Excl2 )";
	  [ ";"; " ) ) and ( not IsEmpty( " ];
	  [ "|"; " & " ]
	) & " ) )"
	= "( not IsEmpty( $Req1 ) ) and ( not IsEmpty( $Req2 ) ) and ( not IsEmpty( ( $Excl1 & $Excl2 ) ) )"

You can see that we have a boolean statement in FileMaker that is checking for required parameters not being empty and for required mutually exclusive parameters having at least one value entered. Using this string in an `Evaluate` function will allow us to make sure that all of the required parameters are present and have values. Obviously, given how this works, empty values can't work for required parameters.

We're almost there. Now we return `True` if one of two conditions is true. In the `SetAssignParamVars` function we had set `$EmptyParamTemplate` to `"( not IsEmpty( $ ) )"`. If all the parameters are optional, then this is what we'll get for the `$FMCode`. If that's the case, we return `True`. The other case is when the `$FMCode`, when evaluated, returns `True`. We store this in a calculation variable `_result`.

    _result = ( $FMCode = $EmptyParamTemplate ) or Evaluate( $FMCode )
            = ( "( not IsEmpty( $Req1 ) ) and ( not IsEmpty( $Req2 ) ) and ( not IsEmpty( ( $Excl1 & $Excl2 ) ) )" = "( not IsEmpty( $ ) )" ) or Evaluate( "( not IsEmpty( $Req1 ) ) and ( not IsEmpty( $Req2 ) ) and ( not IsEmpty( ( $Excl1 & $Excl2 ) ) )" )
            = False or ( ( not IsEmpty( $Req1 ) ) and ( not IsEmpty( $Req2 ) ) and ( not IsEmpty( ( $Excl1 & $Excl2 ) ) ) )
            = False or True
            = True

Since in our example all of the required variables are present, the `Evaluation` function will return true, which means that the entire `AssignParams` function will return true, as at the end, it returns the value of the `_result` calculation variable.

But there's a bit more code before we do that. Basically, once we have this working, we're going to have a lot of script variables hanging around afterwards that will be showing up in the Data Viewer unless we clear them out. Remember that `$Troubleshoot` script variable that was set to `False`? As long as it is set to `False`, the last bit of assignment in `AssignParams` simply nulls out all of the script variables that we used (all 18 of them!). You can turn `$Troubleshoot` to `True` if you want to check out how things work, but once it is working, trust me, leave `$Troubleshoot` as `False` so that when you are debugging scripts, you don't need to see all those temporary script variables clogging up the Data Viewer.

The last thing we do is return the `_result` calculation variable to whoever called `AssignParams` so that the calling script can act accordingly based on whether the required parameters were present or not.

As you may have noticed by now, this isn't perfect. Required parameters can't be blank, and there's not yet any checking to make sure that mutually exclusive parameters actually only get one of the parameters passed. That last one has long been on my todo list, but honestly, it's never really been a problem and so never a priority.

Regardless, I've been using this technique for about almost six years now, and although the function has received minor tweaks since then, it really hasn't changed in functionality and has worked wonderfully over that time. I hope someone finds all of this useful and welcome comments or questions about the technique.
[Matt Petrowski]: https://twitter.com/#!/mattpetrowsky
[function scripting in FileMaker]: http://www.filemakermagazine.com/videos/function-scripting.html
[FileMaker help]: http://www.filemaker.com/12help/html/create_script.13.35.html
[Jesse Antunes]: http://sixfriedrice.com/wp/about/
[Six Fried Rice]: http://sixfriedrice.com/
[Six Fried Rice Article]: http://sixfriedrice.com/wp/passing-multiple-parameters-to-scripts-advanced/
[FileMaker Dictionary Functions]: http://sixfriedrice.com/wp/filemaker-dictionary-functions/
[Thomas Seidler]: http://harmlesswise.com/
[solve the problem]: http://harmlesswise.com/0906-filemaker-python-dictionary-list-functions
[github repository]: https://github.com/chivalry/filemaker-multiple-parameters
[download the technique file directly]: images/articles/funciton_scripting/functionscripting.fp7.zip
[`Evaluate`]: http://www.filemaker.com/12help/html/func_ref3.33.4.html
[`Let`]: http://www.filemaker.com/help/html/func_ref3.33.15.html