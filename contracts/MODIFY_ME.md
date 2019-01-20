# Auction State Machine

In this exercise, we'll set up a simple auction contract. This will guide you through a state machine based contract, which you might find useful represent your contract's behaviour.

## Auction

Let's write an Auction contract that does the following:

First, an auction is _idle_. This means there's nothing to buy or sell yet and the auction is waiting for someone to start selling.

Then someone offers an item to sell. This will be done by calling `startSelling(bytes32 _product, uint initialPrice)`. At this point, the auction goes from the _idle_ state to the _selling_ state and starts collecting offers.

Once the auction is in the _selling_ state, anyone can offer an amount using `offer(uint price)`. This does not cost anything at this point, the buyer will only have to pay if they win the auction.

At any point, the seller (and only the seller) can call `acceptOffer()`. This means that the seller is happy with the highest bid so far and has agreed to sell the product to that bidder. The auction goes from the _selling_ state to the _accepted_ state - noone can offer anymore and the auction waits for the auction winner to pay the promised amount.

Once the offer has been accepted, the highest bidder will have to call `pay()`. This is a payable function so the buyer will have to attach a wei value (`msg.value`) equal to the offered amount. This amount will then be transferred to the seller and the auction is complete.

At this point, the auction can go back to the _idle_ state, waiting for anyone else to start selling their product.

## State machine

Note the state transitions:

Idle -> Selling -> Accepted -> Idle

This can easily be represented by an enum:
```solidity
enum Status { Idle, Selling, Accepted }
```

To enforce the state moves in the right direction, you'll want to write an `atState(Status state)` modifier that requires the contract to be in the correct state for functions like `offer()` or `pay()` to be called.

{% exercise %}
Fill in the blanks to complete the Auction contract.

{% hints %}
- Use `require()` and modifiers to ensure the auction behaves correctly.
- Use `msg.sender` to find out about the seller and buyer's addresses. Which functions can only be called by whom?
- Verify your payment is correct! Check who's paying, how much their paying and ensure the seller receives the payment.

{% initial %}
pragma solidity ^0.4.24;

contract Auction {
    // State machine definition.
    enum Status { Idle, Selling, Accepted }
    Status private status = Status.Idle;
    
    // Keep track of the current state of the auction.
    bytes32 public product = "";
    address private seller = 0;
    uint public highestBid = 0;
    address private highestBidder = 0;
    
    // Ensure the auction is in the correct state
    modifier atState(Status state) {
    }
    
    // Start selling a product with a given initial price.
    function startSelling(bytes32 _product, uint initialPrice) public {
    }

    // Make an offer for the current product on sale. Can only be called with an offer higher than the current highest bid.
    function offer(uint price) public {
    }
    
    // Accept the current highest bid. Can only be called by the seller.
    function acceptOffer() public {
    }
    
    // Pay the agreed price. Can only be called by whoever won the auction.
    function pay() public payable {
    }
    
    // Helper function to reset the auction to its initial state.
    function reset() private {
        product = "";
        seller = 0;
        highestBid = 0;
        highestBidder = 0;
        
        status = Status.Idle;
    }
}

{% solution %}
pragma solidity ^0.4.24;

contract Auction {
    // State machine definition.
    enum Status { Idle, Selling, Accepted }
    Status private status = Status.Idle;
    
    // Keep track of the current state of the auction.
    bytes32 public product = "";
    address private seller = 0;
    uint public highestBid = 0;
    address private highestBidder = 0;
            
    // Ensure the auction is in the correct state
    modifier atState(Status state) {
        require(status == state);
        _;
    }

    modifier sellerOnly(address _address) {
        require (_address == seller);
        _;
    }
    
    modifier buyerOnly(address _address) {
        require (_address == highestBidder);
        _;
    }
    
    modifier higherThanHighest(uint price) {
        require(price > highestBid);
        _;
    }
    
    // Start selling a product with a given initial price.
    function startSelling(bytes32 _product, uint initialPrice) public atState(Status.Idle)  {
        product = _product;
        seller = msg.sender;
        highestBid = initialPrice;
        highestBidder = msg.sender;
        
        status = Status.Selling;
    }

    // Make an offer for the current product on sale. Can only be called with an offer higher than the current highest bid.
    function offer(uint price) public higherThanHighest(price) atState(Status.Selling) {
        highestBid = price;
        highestBidder = msg.sender;
    }
    
    // Accept the current highest bid. Can only be called by the seller.
    function acceptOffer() public sellerOnly(msg.sender) atState(Status.Selling) {
        status = Status.Accepted;
    }
    
    // Pay the agreed price. Can only be called by whoever won the auction.
    function pay() public payable buyerOnly(msg.sender) atState(Status.Accepted) {
        require(msg.value == highestBid);
        
        seller.transfer(msg.value);
        reset();
    }
    
    // Helper function to reset the auction to its initial state.
    function reset() private {
        product = "";
        seller = 0;
        highestBid = 0;
        highestBidder = 0;
        
        status = Status.Idle;
    }
}

{% validation %}
pragma solidity ^0.4.24;

import 'Assert.sol';
import 'Auction.sol';

contract TestAuction {
  Auction auction = Auction(__ADDRESS__);
  
  function testOffer() public {
    auction.startSelling("My Book", 10);
    auction.offer(20);
    Assert.equal(auction.highestBid, 20, "Should remember the highest bid");
  }
  
  function testPayment() public {
    // Seller is the same as buyer for simplicity
    auction.startSelling("My Book", 0);
    uint bid = 1;
    auction.offer(bid);
    auction.acceptOffer();
    auction.pay.value(bid)();
  }

  event TestEvent(bool indexed result, string message);
}

{% endexercise %}

Now you're familiar with modelling state machines in Solidity! This is a very flexible way of representing behaviour which you might find useful when designing your own contracts.

