# Project-Showcase
Portfolio of selected course projects (Java) showcasing backend, data, and UI problem-solving.

## Academic Integrity Note
These are course projects. To prevent plagiarism, this repository only includes high-level descriptions, screenshots, and selected code excerpts. Full source code can be shared privately with employers on request.

## Featured Projects

### 1) WellingtonTrains (Java)

## Demo
- 🎥 Video demo (MP4): https://github.com/MengNiu-3/Project-Showcase/releases/tag/demo-v1


## What I built (highlights)
- Loaded and modelled real timetable data into `Station`, `TrainLine`, and `TrainService` objects.
- Implemented station/line queries (e.g., stations on a line, lines through a station).
- Built a “next services” search that finds upcoming departures at a station after a given time.
- Implemented trip finding with either a direct ride or a single transfer, choosing the earliest arrival.
- Added map visualisation to draw routes on the system map.

## Key design / algorithm
- **Data model**:  
  - `Map<String, Station>` for O(1) station lookup by name  
  - `Map<String, TrainLine>` for O(1) line lookup  
  - each `TrainLine` keeps an ordered list of stations + list of services (each service holds a time list aligned to station indices)

### Direct trip
For each line that contains **both** start and destination (and start index < destination index):
- scan all services on that line
- choose the service with the earliest valid departure (>= query time)
- compare overall by earliest arrival

### Single-transfer trip (my favourite part)
If the destination is **not** on the start line, I search for a trip with exactly one transfer:
1. Choose a sensible transfer station.
2. Find the best service for leg 1 (start → transfer).
3. Find the best service for leg 2 (transfer → destination) that is **catchable** (depart >= arrival at transfer).
4. Compare overall trips by earliest arrival.

---

## Transfer logic (refactor story + why it’s simpler)

When I first implemented transfers, I overcomplicated the problem: because each line has multiple services, I tried to “globally optimise” by comparing many combinations of transfer stations and service pairs. It quickly became hard to reason about, easy to make mistakes, and unfriendly to read.

I deleted that version and restarted with one goal: **make the reasoning explicit and modular**.

After discussing with a tutor and sketching the rail network on paper, I simplified the solution into two clear stages:

1) **Pick the transfer station first**  
I collect candidate transfer stations by finding stations that:
- appear on the current start line, and
- also appear on any line that reaches the destination.

Then I choose the **closest valid transfer station ahead of the start station** on the start line (smallest station index greater than start index).  
I also add a practical preference: if **Wellington** is a valid candidate after the start station, prefer it (common interchange hub).

2) **Match two services by time**  
Once the transfer station is fixed, the timetable search becomes straightforward:
- **Leg 1:** earliest service from start → transfer departing at/after query time
- **Leg 2:** earliest service from transfer → destination that departs at/after arriving at transfer

Why this approach is clearer:
- fewer moving parts at once (choose station, then choose services)
- logic aligns with real-world transfer behaviour
- code becomes easier to test and debug

---

## Code excerpt (core transfer matching)

### 1) Build candidates + pick transfer station

// Transfer station candidates: stations on BOTH the start line and a destination line
Map<String, Station> transferStationCandidates = new HashMap<>();
for (TrainLine destLine : destStation.getTrainLines()) {
    for (Station transferStation : destLine.getStations()) {
        if (stationsOnLine.contains(transferStation)) {
            transferStationCandidates.put(transferStation.getName(), transferStation);
        }
    }
}

// Pick the closest transfer station after the start station (smallest index > startIndex)
for (Station transferS : transferStationCandidates.values()) {
    int idx = stationsOnLine.indexOf(transferS);
    if (idx > startIndex && (bestTransferIdx == -1 || idx < bestTransferIdx)) {
        bestTransferIdx = idx;
    }
}

// Prefer Wellington if it is a valid candidate after the start station
int wellIdx = stationsOnLine.indexOf(stations.get("Wellington"));
if (wellIdx > startIndex && transferStationCandidates.containsKey("Wellington")) {
    bestTransferIdx = wellIdx;
}

if (bestTransferIdx == -1) { continue; }
bestTransferStation = stationsOnLine.get(bestTransferIdx);

### 2) Match services for two legs (catchable connection)
// Leg 1: earliest service from start -> transfer (depart >= query time)
for (TrainService service : line.getTrainServices()) {
    List<Integer> times = service.getTimes();
    int depart = times.get(startIndex);
    int arriveTransfer = times.get(bestTransferIdx);

    if (depart != -1 && arriveTransfer != -1 && depart >= time) {
        if (depart < bestStartTime) {
            bestStartTime = depart;
            arriveTimeAtTransfer = arriveTransfer;
            bestService = service;
        }
    }
}

// Leg 2: earliest catchable service from transfer -> destination (depart >= arriveTransfer)
for (TrainLine transferLine : bestTransferStation.getTrainLines()) {
    if (!transferLine.getStations().contains(destStation)) continue;

    int transferIdx2 = transferLine.getStations().indexOf(bestTransferStation);
    int destIdx2 = transferLine.getStations().indexOf(destStation);
    if (transferIdx2 >= destIdx2) continue;

    for (TrainService service2 : transferLine.getTrainServices()) {
        List<Integer> times2 = service2.getTimes();
        int depart2 = times2.get(transferIdx2);
        int arrive2 = times2.get(destIdx2);

        if (depart2 != -1 && arrive2 != -1 && depart2 >= arriveTimeAtTransfer) {
            if (bestTransferTime == -1 || depart2 < bestTransferTime) {
                bestTransferTime = depart2;
                bestArriveTime = arrive2;
                bestTransferService = service2;
            }
        }
    }
}
## What I learned

### 1) Refactoring is part of problem-solving: simplifying the structure reduced bugs and made the algorithm explainable.
### 2) Build a mental model first: drawing the network clarified constraints (direction, indices, “catchable” transfers).
### 3)Readable logic wins: a solution that teammates can follow and verify is more valuable than a clever but tangled one.
### 4) Ask for help early: discussing roadblocks with a tutor/mentor often unlocks new ideas and faster solutions.

## Tech
Java, Git/GitHub, VS Code, ECS100 library (course framework)


