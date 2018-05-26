<h2>Rest API Manual</h2>

<p>The following <strong>endpoints</strong> are available:
<ul>
  <li><kbd>/</kbd> - list failures (failed jobs and tasks having at least one failed job).</li>
  <li><kbd>/jobs</kbd>  - list non-finished jobs.</li>
  <li><kbd>/tasks</kbd>  - list non-finished tasks.<br />
  <strong>Note</strong>: a task is registered in the <samp>ExecPool</samp> on start of the first descendant job, so only tasks related to the started jobs are shown.</li>
</ul>
</p>

<p>All endpoints have uniform URL <strong>parameters</strong>:
<ul>
  <li>
    <kbd id="fmt">fmt</kbd>  - required format of the result: <strong><samp>json htm txt</samp></strong>
  </li>
  <li>
    <kbd id="cols">cols</kbd>  - item (job/task) properties to be shown (all by default):
    <strong><samp>category rcode duration memkind memsize name numadded numdone numterm pid task tstart tstop</samp></strong>
  </li>
  <li>
    <kbd id="flt">flt</kbd>  - items filtering by the specified properties in the following format:
    <strong><samp>&lt;pname&gt;[*][:&lt;beg&gt;[..&lt;end&gt;]]</samp></strong>
    <ul>
      <li>
        <kbd>&lt;pname&gt;</kbd>  - property name, see <a href="#cols"><kbd>cols</kbd></a>.
      </li>
      <li>
        <kbd>*</kbd>  - optional property, the item is filtered out only if
        the property value is out of the specified range but not if
        this property is not present.
      </li>
      <li>
        <kbd>beg</kbd>  - begin of the property value range, inclusive.
        Exact match is expected if <kbd>end</kbd> is omitted.
      </li>
      <li>
        <kbd>end</kbd>  - end of the property value range, exclusive.
      </li>
    </ul>
    Fully omitted range requires any non-None property value. <br />
    For example, to show terminated jobs with return code <var>-15</var> and tasks having these jobs (<samp>rcode*</samp> is optional since tasks do not have this property), where the jobs have defined <samp>category*</samp> (optional since tasks do not have category) and where each job executed from <var>1.5 sec</var> up to <var>1 hour</var> (3600 sec):
    <samp>?flt=rcode*:-15|duration:1.5..3600|category*</samp>
  </li>
  <li>
    <kbd>jlim</kbd>  - limit the number of the showing items up to this number of jobs, <var>100</var> by default.
  </li>
  <li>
    <kbd>refresh</kbd>  - page auto refresh time in seconds, >= 2, absent by default.
  </li>
</ul>
</p>

<p> An example of the query:
<pre>
http://localhost:8080/tasks?flt=rcode*:-15|duration:1.5..3600|category*
</pre>
</p>