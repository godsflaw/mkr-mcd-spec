```k
requires "kmcd-driver.k"
requires "gem.k"
requires "vat.k"

module FLAP
    imports KMCD-DRIVER
    imports GEM
    imports VAT
```

```k
    syntax Bid ::= FlapBid ( bid: Wad, lot: Rad, guy: Address, tic: Int, end: Int )
 // -------------------------------------------------------------------------------
```

Flap Configuration
------------------

```k
    configuration
      <flap-state>
        <flap-addr>  0:Address    </flap-addr>
        <flap-bids> .Map          </flap-bids>  // mapping (uint => Bid) Int |-> FlapBid
        <flap-kicks> 0            </flap-kicks>
        <flap-live>  true         </flap-live>
        <flap-beg>   105 /Rat 100 </flap-beg>
        <flap-ttl>   3 hours      </flap-ttl>
        <flap-tau>   2 days       </flap-tau>
      </flap-state>
```

Flap Semantics
--------------

```k
    syntax MCDContract ::= FlapContract
    syntax FlapContract ::= "Flap"
    syntax MCDStep ::= FlapContract "." FlapStep [klabel(flapStep)]
 // ---------------------------------------------------------------
    rule contract(Flap . _) => Flap
    rule [[ address(Flap) => ADDR ]] <flap-addr> ADDR </flap-addr>

    syntax FlapStep ::= FlapAuthStep
    syntax AuthStep ::= FlapContract "." FlapAuthStep [klabel(flapStep)]
 // --------------------------------------------------------------------
    rule <k> Flap . _ => exception ... </k> [owise]

    syntax Event ::= FlapKick(Int, Rad, Wad)
 // ----------------------------------------
```

- kick(uint lot, uint bid) returns (uint id)
- Starts a new surplus auction for a lot amount

```k
    syntax FlapAuthStep ::= "kick" Rad Wad
 // --------------------------------------
    rule <k> Flap . kick LOT BID
          => call Vat . move MSGSENDER THIS LOT
          ~> KICKS +Int 1
          ...
         </k>
         <msg-sender> MSGSENDER </msg-sender>
         <this> THIS </this>
         <currentTime> NOW </currentTime>
         <flap-bids>... .Map =>
            KICKS +Int 1 |-> FlapBid(... bid: BID,
                                         lot: LOT,
                                         guy: MSGSENDER,
                                         tic: 0,
                                         end: NOW +Int TAU)
         ...</flap-bids>
         <flap-kicks> KICKS => KICKS +Int 1 </flap-kicks>
         <flap-live> true </flap-live>
         <flap-tau> TAU </flap-tau>
         <frame-events> _ => ListItem(FlapKick(KICKS +Int 1, LOT, BID)) </frame-events>
```

- tick(uint id)
- Extends the end time of the auction if no one has bid yet

```k
    syntax FlapStep ::= "tick" Int [klabel(FlapTick),symbol]
 // --------------------------------------------------------
    rule <k> Flap . tick ID => . ... </k>
         <currentTime> NOW </currentTime>
         <flap-bids>...
           ID |-> FlapBid(... tic: 0, end: END => NOW +Int TAU)
         ...</flap-bids>
         <flap-tau> TAU </flap-tau>
      requires END <Int NOW
```

- tend(uint id, uint lot, uint bid)
- Places a bid made by the user. Refunds the previous bidder's bid.

```k
    syntax FlapStep ::= "tend" Int Rad Wad
 // --------------------------------------
    rule <k> Flap . tend ID LOT BID
          => call Gem "MKR" . move MSGSENDER GUY BID'
          ~> call Gem "MKR" . move MSGSENDER THIS (BID -Rat BID')
         ...
         </k>
         <msg-sender> MSGSENDER </msg-sender>
         <this> THIS </this>
         <currentTime> NOW </currentTime>
         <flap-bids>...
           ID |-> FlapBid(... bid: BID' => BID,
                              lot: LOT',
                              guy: GUY => MSGSENDER,
                              tic: TIC => TIC +Int TTL,
                              end: END)
         ...</flap-bids>
         <flap-live> true </flap-live>
         <flap-ttl> TTL </flap-ttl>
         <flap-beg> BEG </flap-beg>
      requires (TIC >Int NOW orBool TIC ==Int 0)
       andBool END  >Int NOW
       andBool LOT ==Rat LOT'
       andBool BID  >Rat BID'
       andBool BID >=Rat BID' *Rat BEG
```

- deal(uint id)
- Settles an auction, rewarding the lot to the highest bidder and burning their bid

```k
    syntax FlapStep ::= "deal" Int [klabel(FlapDeal),symbol]
 // --------------------------------------------------------
    rule <k> Flap . deal ID
          => call Vat . move THIS GUY LOT
          ~> call Gem "MKR" . burn THIS BID
         ...
         </k>
         <this> THIS </this>
         <currentTime> NOW </currentTime>
         <flap-bids>...
           ID |-> FlapBid(... bid: BID, lot: LOT, guy: GUY, tic: TIC, end: END) => .Map
         ...</flap-bids>
         <flap-live> true </flap-live>
      requires TIC =/=Int 0
       andBool (TIC <Int NOW orBool END <Int NOW)
```

- cage(uint rad)
- Part of Global Settlement. Freezes the auction house.

```k
    syntax FlapAuthStep ::= "cage" Rad
 // ----------------------------------
    rule <k> Flap . cage RAD => call Vat . move THIS MSGSENDER RAD ... </k>
         <msg-sender> MSGSENDER </msg-sender>
         <this> THIS </this>
         <flap-live> _ => false </flap-live>
```

- yank(uint id)
- Part of Global Settlement. Refunds the highest bidder's bid.

```k
    syntax FlapStep ::= "yank" Int [klabel(FlapYank),symbol]
 // --------------------------------------------------------
    rule <k> Flap . yank ID
          => call Gem "MKR" . move THIS GUY BID
         ...
         </k>
         <this> THIS </this>
         <flap-bids>...
           ID |-> FlapBid(... bid: BID, guy: GUY) => .Map
         ...</flap-bids>
         <flap-live> false </flap-live>
```

```k
endmodule
```
