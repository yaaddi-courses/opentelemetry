# OpenTelemetry

The vendor-neutral standard for traces, metrics, and logs — from a single span to a full Collector pipeline in production. The final deck (running OTel on Kubernetes) goes smoothly if you already know pods and Deployments — see this app's Kubernetes Fundamentals course first if those are new.

Part of the [Yaaddi](https://github.com/yaaddi-courses) course catalog — a
spaced-repetition flashcard course, ready to build and validate with the
standard Yaaddi course tooling.

## Structure

- `meta.json` — course metadata (title, description, cover image, version)
- `source/` — authoring source (`meta.csv`, `units.csv`, `cards.csv`, images/media)
- the built `.zip` — generated from `source/` via `build_course_zip.py`

## Editing this course

```bash
python validate_course.py . --source
python build_course_zip.py .
```

See [yaaddi-courses/course-template](https://github.com/yaaddi-courses/course-template)
for the full authoring guide (`AUTHORING.md`) and card-format reference.

## License

Course content in this repo is licensed under
[PolyForm Noncommercial 1.0.0](LICENSE).
