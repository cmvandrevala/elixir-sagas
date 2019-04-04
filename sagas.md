# The Saga Pattern

## Let's Build a Travel Booking Website!

It will be the greatest website the world has ever seen. :airplane:

## We Start Off With the Happy Path

```elixir
def book_trip() do
  book_flight()
  book_hotel()
  book_car()
  charge_credit_card()
  IO.inspect "SUCCESS!"
end
```

Many of these functions depend on third party APIs. What if one of them fails for some reason?

## Let’s Add in Some Conditional Logic

We only want to progress to the next step if the previous step succeeds.

 ```elixir
 def book_trip() do
  flight_booked = book_flight()
  if flight_booked do
    hotel_booked = book_hotel()
    if hotel_booked do
      car_booked = book_car()
      if rental_car_booked do
        charge_credit_card()
        IO.inspect "SUCCESS!"
      end
    end
  end
end
```

This works if everything proceeds along the happy path. However, handling failures becomes a pain in the butt...

```elixir
def book_trip() do
 flight_booked = book_flight()
 if flight_booked do
   hotel_booked = book_hotel()
   if hotel_booked do
     car_booked = book_car()
     if rental_car_booked do
       card_charged = charge_credit_card()
       if card_charged do
         IO.inspect "SUCCESS!"
       else
         cancel_car()
         cancel_hotel()
         cancel_flight()
       end
     else
       cancel_hotel()
       cancel_flight()
     end
   else
     cancel_flight()
   end
 end
end
```

Kill it with :fire:

## But We Can Use the With Statement!

The problem with the `if/else` statement approach is that every function in the chain needs to handle the case where any function before it returned an error.

So you say, "Cyrus, nobody uses `if/else` in Elixir. All of the cool kids are using `with` statements!" For example, you can take this code snippet.

```elixir
defmodule User do
  defstruct name: nil, dob: nil

  def create(params) do
    %User{}
    |> parse_dob(params["dob"])
    |> parse_name(params["name"])
  end

  defp parse_dob(user, nil), do: {:error, "dob is required"}
  defp parse_dob(user, dob) when is_integer(dob), do: %{user | dob: dob}
  defp parse_dob(_user, _invalid), do: {:error "dob must be an integer"}

  defp parse_name(_user, {:error, _} = err), do: err
  defp parse_name(user, nil), do: {:error, "name is required"}
  defp parse_name(user, ""), do: parse_name(user, nil)
  defp parse_name(user, name), do: %{user | name: name}
end
```

And transform it into this code snippet:

```elixir
defmodule User do
  defstruct name: nil, dob: nil

  def create(params) do
    with {:ok, dob} <- parse_dob(params["dob"]),
         {:ok, name} <- parse_name(params["name"])
    do
      %User{dob: dob, name: name}
    else
      err -> err
    end
  end

  defp parse_dob(nil), do: {:error, "dob is required"}
  defp parse_dob(dob) when is_integer(dob), do: {:ok, dob}
  defp parse_dob(_invalid), do: {:error "dob must be an integer"}

  defp parse_name(nil), do: {:error, "name is required"}
  defp parse_name(""), do: parse_name(nil)
  defp parse_name(name), do: {:ok, name}
end
```

Example courtesy of the [Open My Mind Blog](https://www.openmymind.net/Elixirs-With-Statement/).

Alright, let's give it a shot. The happy path looks simple enough:

```elixir
def book_trip() do
  with {:ok, _flight} <- FlightAPI.book_flight(),
       {:ok, _hotel} <- HotelAPI.book_hotel(),
       {:ok, _car} <- CarAPI.book_car() do
    BillingAPI.charge_card()
    IO.inspect "SUCCESS!"
  end
end
```

However, we need to handle the errors of each of the assertions in the `with` statement separately since the behavior is different for each error. We need to label each error with an additional atom so that we know what action to take.

```elixir
def book_trip() do
  with {:flight, {:ok, _flight}} <- {:flight, FlightAPI.book_flight()},
       {:hotel, {:ok, _hotel}} <- {:hotel, HotelAPI.book_hotel()},
       {:car, {:ok, _car}} <- {:car, CarAPI.book_car()},
       {:bill, {:ok, _bill}} <- {:bill, BillingAPI.charge_card()} do
    IO.inspect "SUCCESS!"
  else
    {:flight, {:error, reason}} ->
      {:error, reason}

    {:hotel, {:error, reason}} ->
      FlightAPI.cancel_flight()
      {:error, reason}

    {:car, {:error, reason}} ->
      FlightAPI.cancel_flight()
      HotelAPI.cancel_hotel()
      {:error, reason}

    {:bill, {:error, reason}} ->
      FlightAPI.cancel_flight()
      HotelAPI.cancel_hotel()
      CarAPI.cancel_car()
      {:error, reason}
  end
```

Kill it with :fire:

## Is All Hope Lost?

No

## Enter Sagas

The idea of sagas came from a [research paper from 1987](http://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf) dealing with long-lived transactions. Quote:

> A long lived transaction is a Saga if it can be written as a sequence of transactions that can be interleaved with other transactions.

> The database management system guarantees that either all the transactions in a Saga are successfully completed or compensating transactions are run to amend a partial execution.

A diagram makes this much clearer:

![Saga Diagram](/saga.jpeg)
Image Courtesy of [Andrew Dryga](https://medium.com/nebo-15/introducing-sage-a-sagas-pattern-implementation-in-elixir-3ad499f236f6)

Compensations undo the transaction effects. They are not always perfect inverses! For example:

* If you book a room, you might be able to un-book it. However, maybe the company will incur a service charge.
* If you sent an email confirmation you can't unsend it. However,  you can send a follow up email with an apology.

## The Sage Package

The [sage package](https://hex.pm/packages/sage) creates some really nice scaffolding if you want to implement sagas. Notice how there is a bit of boilerplate needed to get it running, but the business logic is no longer repeated across multiple edge cases.

```elixir
defmodule MyApp.Sage do
  import Sage

  def book_trip(attributes) do
    new()
    |> run(:flight, &book_flight/2, &cancel_flight/4)
    |> run(:hotel, &book_hotel/2, &cancel_hotel/4)
    |> run(:car, &book_car/2, &cancel_car/4)
    |> finally(:charge, &charge_card/2)
    |> transaction(Repo, attributes)
  end

  def book_flight(_effects_so_far, _attrs) do
    :ok
  end

  def cancel_flight(_effect_to_compensate, _effects_so_far, _name_and_reason, _attrs) do
    :abort
  end

  # And so on...
  # There are lots of options for the callbacks.
  # Check out the docs.
end
```

## Wow, This Sounds Great! I am Going to Use it Everywhere!

Calm down, there. It is pretty neat, but it does have a BIG downside.

## The ACID Acronym

* Atomic: Either all of the changes in a transaction are made (commit) or none are made (rollback).
* Consistent: A transaction won't violate the declared system integrity constraints (e.g. indices, keys). In other words, the database is valid before and after the transaction.
* Isolated: Transaction results are unaffected by other concurrent transactions.
* Durable: Committed changes survive a hardware failure.

By definition, sagas cannot be isolated.

However, this tradeoff can be reasonable in a few cases:

1. Long-Running Transactions That Lock the Database
1. Distributed Transactions Across Hundreds of Microservices
1. Processes That Are Well-Defined by State Machines
1. Processes Where the ACID Requirement is Not Necessary

## Final Thoughts

* Long-Running or Distributed Transactions Might Require a Change of Architecture to Keep Code Clean
* The Saga Pattern is One Way to Handle These Types of Transactions
* However, the Tradeoff You Make is Loss of Isolation (ACID)
* Sage is Kind of Neat

## Questions?
