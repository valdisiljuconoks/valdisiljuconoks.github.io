---
title: Matching the Patterns
author: valdis
date: 2013-09-09 23:45:00 +0200
categories: [.NET, F#]
tags: [.net, f#]
---

Cna yuo read this?

Aiccdrong to recent sefiinctic resarech fro us to be able to read teh text teh only ipmort tihng is that first adn teh last ltteer in a word aer in teh rgiht place. Teh rest of teh letetrs aucatlly does nto matter. It could be really teh gtearest of teh mssees adn we still would be able to read teh text. Your barin R-mdoe engine is able to mtach teh petatrn bsaed in teh meerst famegrnt of teh ptetarn in irneetst. Even text is taltoly msseed up yuo cna still read wohtiut a pbrloem. This is psibosle bceuase hmuan mind is nto riedang eevry letter by iestlf btu a word as wohle. It means that riadeng speed cna be insacered even more by tkiang pcicrtaes to rieongcze prteatns in wrods fstaer adn nto to read eevry single ltteer.

I find it rlaely aiamzng!

By teh wya code bleow wsa used as brain pictarce uinsg rcieursve to sfufhle this text so yuo cna mtach pattrens adn nto read each lteter in word btu word isletf:

```fsharp
open System
open System.Linq

let marks = [","; "."; "?"; "!"; "?!"]
let rnd = new Random()

let rec insert v i l =
    match i, l with
    | 0, h -> v::h
    | i, h::t -> h::insert v (i - 1) t
    | _, [] -> failwith "index out of range"

let rec remove i l =
    match i, l with
    | 0, h::t -> t
    | i, h::t -> h::remove (i - 1) t
    | _, [] -> failwith "index out of range"

let shuffleWord (input:string) =
    let result = Array.create input.Length "00"

    let rec _shuffle (inp:list<char>) ix =
        match ix with
        | 0 -> [inp.[0]]
        | _ ->
            let idx = rnd.Next(inp.Length - 1)
            inp.[idx] :: _shuffle (remove idx inp) (ix - 1)

    let w =
        match input.Length with
        | l when l = 1 -> [input.[0]]
        | l when l < 3 -> [input.[0]; input.[1]]
        | l when l = 3 -> [input.[0]; input.[2]; input.[1]]
        | _ ->
            let midPart = input.[1 .. input.Length - 2] |> List.ofSeq
            input.[0] :: (_shuffle midPart (midPart.Length - 1 )) @ [input.Last()]

    w
    |> List.map (fun (c:char) -> c.ToString())
    |> List.reduce (+)

let rec shuffle (input:string) =
    let rec _clear (input:string) marks =
        match marks with
        | [] -> [input]
        | h::t ->
            if input.EndsWith(h) then (input.Substring(0, input.Length - 1)) :: [h]
            else _clear input t

    let rec _shuffle list =
        match list with
        | [] -> []
        | h::t -> ((_clear h marks) |> List.map shuffleWord) @ _shuffle t

    let arrayOfWords = (input.Split([|" "|], StringSplitOptions.None) |> List.ofArray)
    _shuffle arrayOfWords |> List.reduce (fun a b -> a + " " + b)


> shuffle "By the way code below was used as brain practice to shuffle this text."
```

[*eof*]
