## Index page
{% raw %}
```c++
const std::string* cmMakefile::GetDef(const std::string& name) const
{
  cmProp def = this->StateSnapshot.GetDefinition(name);
  if (!def) {
    def = this->GetState()->GetInitializedCacheValue(name);
  }
#ifndef CMAKE_BOOTSTRAP
  cmVariableWatch* vv = this->GetVariableWatch();
  if (vv && !this->SuppressSideEffects) {
    bool const watch_function_executed =
      vv->VariableAccessed(name,
                           def ? cmVariableWatch::VARIABLE_READ_ACCESS
                               : cmVariableWatch::UNKNOWN_VARIABLE_READ_ACCESS,
                           (def ? def->c_str() : nullptr), this);

    if (watch_function_executed) {
      // A callback was executed and may have caused re-allocation of the
      // variable storage.  Look it up again for now.
      // FIXME: Refactor variable storage to avoid this problem.
      def = this->StateSnapshot.GetDefinition(name);
      if (!def) {
        def = this->GetState()->GetInitializedCacheValue(name);
      }
    }
  }
#endif
  return def;
}
```
{% endraw %}
