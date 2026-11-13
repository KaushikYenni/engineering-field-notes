# Reducing temporary-file noise without losing useful signals

At Microsoft I worked on a feature to reduce noise from temporary-file
activity. The hard part was not simply filtering out activity; it was making
the noise smaller without hiding signals that mattered.

I led the design and delivery of targeted suppression. We looked for the
dominant noise patterns and tried to keep the change narrow, so useful signals
about what people did with data stayed visible.

The judgment call was in the boundary: suppress too little and the noise
remains; suppress too much and the product loses context. That balance is
what I remember most about the work.
