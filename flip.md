```k
requires "kmcd-driver.k"
requires "vat.k"

module FLIP
    imports KMCD-DRIVER
    imports VAT
```

```k
    syntax Bid ::= FlipBid ( bid: Rad, lot: Wad, guy: Address, tic: Int, end: Int, usr: Address, gal: Address, tab: Rad )
 // ---------------------------------------------------------------------------------------------------------------------
```

Flip Configuration
------------------

```k
    configuration
      <flips>
        <flip multiplicity="*" type="Map">
          <flip-ilk>   ""           </flip-ilk>
          <flip-addr>  0:Address    </flip-addr>
          <flip-bids>  .Map         </flip-bids> // mapping (uint => Bid)     Int     |-> FlipBid
          <flip-beg>   105 /Rat 100 </flip-beg>  // Minimum Bid Increase
          <flip-ttl>   3 hours      </flip-ttl>  // Single Bid Lifetime
          <flip-tau>   2 days       </flip-tau>  // Total Auction Length
          <flip-kicks> 0            </flip-kicks>
        </flip>
      </flips>
```

Flip Semantics
--------------

```k
    syntax MCDContract ::= FlipContract
    syntax FlipContract ::= "Flip" String
    syntax MCDStep ::= FlipContract "." FlipStep [klabel(flipStep)]
 // ---------------------------------------------------------------
    rule contract(Flip ILK . _) => Flip ILK
    rule [[ address(Flip ILK) => ADDR ]] <flip-ilk> ILK </flip-ilk> <flip-addr> ADDR </flip-addr>

    syntax FlipStep ::= FlipAuthStep
    syntax AuthStep ::= FlipContract "." FlipAuthStep [klabel(flipStep)]
 // --------------------------------------------------------------------
    rule <k> Flip _ . _ => exception ... </k> [owise]

    syntax Event ::= FlipKick(Int, Wad, Rad, Rad, Address, Address)
 // ---------------------------------------------------------------

    syntax FlipStep ::= "kick" Address Address Rad Wad Rad
 // ------------------------------------------------------
    rule <k> Flip ILK . kick USR GAL TAB LOT BID
          => call Vat . flux ILK MSGSENDER THIS LOT
          ~> KICKS +Int 1 ... </k>
         <msg-sender> MSGSENDER </msg-sender>
         <this> THIS </this>
         <currentTime> NOW </currentTime>
         <flip-ilk> ILK </flip-ilk>
         <flip-tau> TAU </flip-tau>
         <flip-kicks> KICKS => KICKS +Int 1 </flip-kicks>
         <flip-bids>... .Map =>
           KICKS +Int 1 |-> FlipBid(...
                             bid: BID,
                             lot: LOT,
                             guy: MSGSENDER,
                             tic: 0,
                             end: NOW +Int TAU,
                             usr: USR,
                             gal: GAL,
                             tab: TAB)
         ...</flip-bids>
         <frame-events> _ => ListItem(FlipKick(KICKS +Int 1, LOT, BID, TAB, USR, GAL)) </frame-events>

    syntax FlipStep ::= "tick" Int
 // ------------------------------
    rule <k> Flip ILK . tick ID => . ... </k>
         <currentTime> NOW </currentTime>
         <flip-ilk> ILK </flip-ilk>
         <flip-tau> TAU </flip-tau>
         <flip-bids>...
           ID |-> FlipBid(... tic: TIC, end: END => NOW +Int TAU)
         ...</flip-bids>
      requires END  <Int NOW
       andBool TIC ==Int 0

    syntax FlipStep ::= "tend" Int Wad Rad
 // --------------------------------------
    rule <k> Flip ILK . tend ID LOT BID
          => call Vat . move MSGSENDER GUY BID'
          ~> call Vat . move MSGSENDER GAL (BID -Rat BID') ... </k>
         <msg-sender> MSGSENDER </msg-sender>
         <currentTime> NOW </currentTime>
         <flip-ilk> ILK </flip-ilk>
         <flip-beg> BEG </flip-beg>
         <flip-ttl> TTL </flip-ttl>
         <flip-bids>...
           ID |-> FlipBid(... bid: BID' => BID,
                              lot: LOT',
                              guy: GUY => MSGSENDER,
                              tic: TIC => NOW +Int TTL,
                              end: END,
                              gal: GAL,
                              tab: TAB)
         ...</flip-bids>
      requires GUY =/=K 0
       andBool (TIC >Int NOW orBool TIC ==Int 0)
       andBool END >Int NOW
       andBool LOT ==Rat LOT'
       andBool BID <=Rat TAB
       andBool BID >Rat BID'
       andBool (BID >=Rat BEG *Rat BID' orBool BID ==Rat TAB)

    syntax FlipStep ::= "dent" Int Wad Rad
 // --------------------------------------
    rule <k> Flip ILK . dent ID LOT BID
          => call Vat.move MSGSENDER GUY BID
          ~> call Vat.flux ILK THIS USR (LOT' -Rat LOT) ... </k>
         <msg-sender> MSGSENDER </msg-sender>
         <this> THIS </this>
         <currentTime> NOW </currentTime>
         <flip-ilk> ILK </flip-ilk>
         <flip-beg> BEG </flip-beg>
         <flip-ttl> TTL </flip-ttl>
         <flip-bids>...
           ID |-> FlipBid(... bid: BID',
                              lot: LOT' => LOT,
                              guy: GUY => MSGSENDER,
                              tic: TIC => NOW +Int TTL,
                              end: END,
                              usr: USR,
                              tab: TAB)
         ...</flip-bids>
      requires GUY =/=K 0
       andBool (TIC >Int NOW orBool TIC ==Int 0)
       andBool END >Int NOW
       andBool BID ==Rat BID'
       andBool BID ==Rat TAB
       andBool LOT <Rat LOT'
       andBool BEG *Rat LOT <Rat LOT'

    syntax FlipStep ::= "deal" Int
 // ------------------------------
    rule <k> Flip ILK . deal ID => call Vat . flux ILK THIS GUY LOT ... </k>
         <this> THIS </this>
         <currentTime> NOW </currentTime>
         <flip-ilk> ILK </flip-ilk>
         <flip-bids>...
           ID |-> FlipBid(... lot: LOT, guy: GUY, tic: TIC, end: END) => .Map
         ...</flip-bids>
      requires TIC =/=Int 0
       andBool (TIC <Int NOW orBool END <Int NOW)

    syntax FlipAuthStep ::= "yank" Int
 // ----------------------------------
    rule <k> Flip ILK . yank ID
          => call Vat . flux ILK THIS MSGSENDER LOT
          ~> call Vat . move MSGSENDER GUY BID ... </k>
         <msg-sender> MSGSENDER </msg-sender>
         <this> THIS </this>
         <flip-ilk> ILK </flip-ilk>
         <flip-bids>...
           ID |-> FlipBid(... bid: BID, lot: LOT, guy: GUY, tab: TAB) => .Map
         ...</flip-bids>
      requires GUY =/=K 0 andBool BID <Rat TAB
```

```k
endmodule
```
