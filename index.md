# Some Thoughts and Questions on Angular's Upcoming Usage of Signals

## State Signals   

IMHO, people who already used state management solutions might prefer this style of using Signals:

```typescript
type ComponentState = {
  from: string;
  to: string;
  urgent: boolean;
  flights: Flight[];
  basket: Record<number, boolean>;
};

const initState: ComponentState = {
  from: 'Hamburg',
  to: 'Graz',
  urgent: false,
  flights: [], 
  basket: {
    3: true,
    5: true
  }
}

@Component({ ... })
export class FlightSearchComponent implements OnInit {
  
  state = signal(initState);

  async search(): Promise<void> {
    if (!this.state().from || !this.state().to) return;

    const flights = await this.flightService.findAsPromise(
        this.state().from, 
        this.state().to,
        this.state().urgent);

    this.state.mutate(s => {
        s.flights = flights;
    });
  }

}
```

As this example shows, this also works well with ``async/await`` and ``Promises``.

Remark: I really need to get used to the usage of parenthesis in statements like ``this.state().from``. 

- Q1: What do you think about this usage of Signals?
- Q2: Is there a way to get rid of the parenthesis? I can really imagine people start writing their own Proxy-based wrappers. Or is it the plan to delegate such opinionated "streamlinings" to vendors of State Management libs?
- Q3: Am I a bad community member when using ``Promises`` and ``mutate`` i/o ``Observables`` and ``update``?

## Flux

A Flux-style State Service ("Service with Signals") could look like this:

```typescript
type ComponentState = {
  from: string;
  to: string;
  urgent: boolean;
  flights: Flight[];
  basket: Record<number, boolean>;
};

const initState: ComponentState = {
  from: 'Hamburg',
  to: 'Graz',
  urgent: false,
  flights: [],
  basket: {
    3: true,
    5: true,
  },
};

@Injectable({ providedIn: 'root' })
export class FlightSearchFacade {
  private flightService = inject(FlightService);

  #state = signal(initState);
  readonly state = this.#state as Signal<ComponentState>; // Not writeable anymore

  async load(): Promise<void> {
    const flights = await this.flightService.findAsPromise(
      this.#state().from,
      this.#state().to,
      this.#state().urgent
    );

      this.#state.mutate((s) => {
        s.flights = flights;
      });
  }
  
  patch(state: Partial<ComponentState>): void {
    // Validate incoming state here
    this.#state.update(s => ({ ...s, ...state }));
  }

  delay(): void {
    this.#state.mutate((s) => {
      const flight = s.flights[0];
      flight.date = addMinutes(flight.date, 15);
    });
  }
}
```

The easiest way of consuming this signal seems to be assigning it to a property in the component:

```typescript
export class FlightSearchComponent implements OnInit {

  private facade = inject(FlightSearchFacade)
  state = this.facade.state;

  [...]
}
```

- Q4: What do you think about this usage?
- Q5: What do you think about the idea of casting from ``SettableSignale`` to ``Signal``? (ofc, the consumer could "cast back" if they are bold enough).


# Flux and Combining Signals

Now, let's assume the state is spread across several signals. Perhaps there is one signal with a piece of global state and another one with local state.

To demonstrate this, let's assume our state service is just returning the flights:

```typescript
@Injectable({providedIn: 'root'})
export class FlightSearchFacade {
    constructor() { }

    flightService = inject(FlightService);
    flights = signal<Flight[]>([]);

    async load(from: string, to: string, urgent: boolean) {
        const flights = await this.flightService.findAsPromise(from, to, urgent);
        this.flights.set(flights);
    } 
}
```

The components has a signal for the overall state:

```typescript
export class FlightSearchComponent implements OnInit {

  private facade = inject(FlightSearchFacade);

  state = signal({
    from: 'Hamburg',
    to: 'Graz',
    urgent: false,
    flights: [] as Flight[], 
    basket: {
      3: true,
      5: true
    } as Record<string, boolean> 
  });

  [...]

}
```

In this case, we need to wire up the flights signal provided by the state service with the flights property in the component:

```typescript
  constructor() {

    effect(() => {
      this.state.mutate(s => {
        s.flights = this.facade.flights()
      });
    });
  }
```

- Q6: What do you think about this usage?
- Q7: Is there a better way of integrating a signal into another one?


## Nesting Signals

One solution for dealing with the situation in the previous section is nesting Signals. It could look like this:

```typescript
export class FlightSearchComponent implements OnInit {

  private facade = inject(FlightSearchFacade);
  private route = inject(ActivatedRoute);

  state = signal({
    from: 'Hamburg',
    to: 'Graz',
    urgent: false,
    flights: this.facade.flights, // Signal in a Signal?
    basket: {
      3: true,
      5: true
    } as Record<string, boolean> 
  });

  [...]
}
```

While this works, the consumer needs to know which parts of the state are signals and which other parts are ordinary properties. For instance, when binding ``flights``, the consumer needs to go with parenthesis; when binding other properties like ``to``, parenthesis are not needed:

```html
<div>Flights to {{state().to}}</div>
<div *ngFor="let f of state().flights()"> ... </div>
```

- Q8: What do you think about this usage?
- Q9: Is it valid to nest signals?
- Q10: Is this something that is delegated to vendors of state management solutions?


## Having Several Signals

Another solution for preventing the need of combining signals is to have several signals, e. g. one per property we want to bind:

```typescript
export class FlightSearchComponent implements OnInit {

  private facade = inject(FlightSearchFacade);

  from = signal('Hamburg');
  to = signal('Graz');
  flights = this.facade.flights;
  basket = signal<Record<number, boolean>>({ 1: true });
  urgent = signal(false);

  [...]
}
```

This could also be a good strategy for beginners that don't know much about state management. It's straight forward: Everything you want to data bind becomes a signal?

- Q11: What do you think about this usage?
- Q12: Could it be a good idea to teach this style in a beginners class while moving to a state signal in an advanced class?

