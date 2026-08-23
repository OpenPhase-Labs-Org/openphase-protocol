# Contributors

OpenPhase is developed and maintained by the contributors listed below. Order is roughly by chronological involvement; alphabetical thereafter. Add yourself in the appropriate section when your first PR merges.

If your name should appear here and doesn't, open a PR adding it.

## Maintainers

- **OpenPhase Labs, LLC** — Protocol stewardship, project governance, releases. Maintains OpenPhase under license from Heritage Grid, LLC. Contact via the OpenPhase Labs governance mailbox.

## IP holder

- **Heritage Grid, LLC** — Copyright holder. Owns the OpenPhase IP and licenses it to OpenPhase Labs, LLC for project operation. See [LICENSE](LICENSE) for terms.

## Authors

- *Initial protocol design, IHR/NTCIP/J2735 schema mapping, gRPC ingestion contract, reference documentation.*

## Contributors

- *Add yourself here on your first merged PR. Format: `**Name** (GitHub @handle, optional affiliation) — short description of contribution(s)`.*

---

## Acknowledgements

OpenPhase builds on the published work of:

- **Sturdevant, J. R., T. Overman, E. Raamot, R. Deer, D. Miller, D. M. Bullock, C. M. Day, T. M. Brennan Jr., H. Li, A. Hainen, and S. M. Remias** (2012). *Indiana Traffic Signal Hi Resolution Data Logger Enumerations*. Indiana Department of Transportation and Purdue University. [doi:10.4231/K4RN35SH](https://doi.org/10.4231/K4RN35SH). The `EventType` enum in `ihr_events.proto` is a direct rendering of this enumeration.
- **AASHTO / ITE / NEMA Joint Committee on the NTCIP**, *NTCIP 1202 v03A — Object Definitions for Actuated Signal Controllers (ASC) Interface Protocol* (May 2019). The OID-addressed messages in `ntcip.proto` mirror this MIB.
- **SAE International**, *SAE J2735 — Dedicated Short Range Communications (DSRC) Message Set Dictionary*. The `MovementEvent` and `MovementState` semantics in `spat.proto` follow the J2735 SPaT message structure.
- **NEMA TS 2** — referenced throughout NTCIP 1202 for parameter semantics and timing definitions.

These are the source standards OpenPhase is faithful to. Discrepancies between the protos and these specs should be filed as bugs.
