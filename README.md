### Deallocation exception handling:

The funcion `lists:keysearch()` in the following code snippet:

```erlang
deallocate({Free, Allocated}, Freq) ->
   {value,{Freq,Pid}} = lists:keysearch(Freq,1,Allocated)
```
will return `false` if the frequency is not found in the Allocated list, i.e. when the client tries to deallocate a frequency that it does not own.  That will cause a bad match error:

```erlang
deallocate({Free, Allocated}, Freq) ->
   {value,{Freq,Pid}} = false   <<****BADMATCH****
```

To catch the badmatch error, I added a try-catch to the following clause in the server's receive loop:

```erlang

        {request, Pid , {deallocate, Freq}} ->
            NewFrequencies = 
                try deallocate(Frequencies, Freq)
                catch
                    error:{badmatch, _Val} -> Frequencies
                end,
            Pid ! {reply, ok},
            loop(NewFrequencies, Supervisor);
 ```
 In that version, the server does nothing when the client tries to deallocate a frequency that it does not own.  
 
 Another approach would be to send an error message to the client:
 
 ```
         {request, Pid , {deallocate, Freq}} -> 
            try deallocate(Frequencies, Freq) of
                NewFrequencies -> 
                    Pid ! {reply, ok},
                    loop(NewFrequencies, Supervisor)
            catch
                error:{badmatch, _Val} ->
                    Pid ! {reply, {error, not_allocated}},
                    loop(Frequencies, Supervisor)
            end;
``` 
But, as was mentioned in the lecture, then the client would have to add machinery to handle the error message.

I don't really see why killing the client would be necessary when the client tries to deallocate a frequency it doesn't own, but another approach would be for the server to kill the client:
 

```erlang
        {request, Pid , {deallocate, Freq}} -> 
            try deallocate(Frequencies, Freq) of
                NewFrequencies -> 
                    Pid ! {reply, ok},
                    loop(NewFrequencies, Supervisor)
            catch
                error:{badmatch, _Val} ->
                    exit(Pid, not_allocated),
                    io:format("client (~w) deallocated bad frequency: ~w...killed client~n", [Pid, Freq]),
                    loop(Frequencies, Supervisor)
            end;
 ```

### Unknow Message exception handling:

I added a new clause to the bottom of the server's receive loop:

```erlang
        Other ->
            try handle(Other)
            catch
                throw: {unknown_request, Val} -> 
                    io:format("server (~w) got unknown request: ~w...discarding~n", [self(), Val]),
                    loop(Frequencies, Supervisor)
            end
    

handle(Other) ->
    throw({unknown_request, Other}).
    
```
I felt that was a pretty contrived solution, and I couldn't think of any alternate approaches.
