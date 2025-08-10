# T-Cell-Movement-Simulation-in-Lymph-Nodes
; ===== T Cell Movement Within Lymph Nodes - Enhanced Model =====
; Author: M.C.H. Pinto (D/BCS/23/0015)
; Course: CS3142 - Complex Systems and Agent Technology
; Instructor: Dr. Kaneeka Vidanage

breed [tcells tcell]
breed [DCs DC]

globals [
  dt              ; minutes per time step
  ds              ; micrometers per patch  
  tcell-step      ; patches per time step for T cells
  nDCmeet         ; total DC-T cell encounters
  nMeetDC1        ; unique T cells meeting DC1
  nMeetDC2        ; unique T cells meeting DC2  
  nMeetBoth       ; unique T cells meeting both DCs
]

tcells-own [
  met1?           ; has this T cell met DC1?
  met2?           ; has this T cell met DC2?
]

links-own [
  tDCfree         ; minutes remaining for current contact
]

patches-own [
  chemokine       ; chemokine concentration (for chemotaxis extension)
]

; ========================= SETUP =========================
to setup
  clear-all
  set-default-shape turtles "circle"
  
  ; Create T cells
  create-tcells 100 [
    set color green
    set size 1
    setxyz random-xcor random-ycor random-zcor
    set met1? false
    set met2? false
  ]
  
  ; Create dendritic cells
  create-DCs 2 [
    set size 2
    setxyz random-xcor random-ycor random-zcor
  ]
  
  ; Assign distinct colors to DCs (red = DC1, orange = DC2)
  let dc-list sort DCs
  if length dc-list >= 1 [ ask item 0 dc-list [ set color red ] ]
  if length dc-list >= 2 [ ask item 1 dc-list [ set color orange ] ]
  
  ; Set biological parameters
  set dt 0.5                    ; 0.5 min per time step
  set ds 8                      ; 8 μm per patch
  set tcell-step 16 / ds * dt   ; 16 μm/min → patches/step
  
  ; Initialize counters
  set nDCmeet 0
  set nMeetDC1 0
  set nMeetDC2 0
  set nMeetBoth 0
  
  reset-ticks
end

; ========================= MAIN LOOP =========================
to go
  if chemokine-mode? [
    secrete-chemokines
    diffuse-chemokines
  ]
  
  move-tcells
  identify-DCbound-tcells
  update-link-status
  
  ; Update plotting
  plotxy ticks nDCmeet
  
  tick-advance dt
end

; ======================= MOVEMENT =======================
to move-tcells
  ask tcells [
    if color != blue [  ; only move if not bound
      ifelse chemokine-mode? [
        chemokine-biased-walk
      ] [
        persistent-random-walk
      ]
    ]
  ]
end

to persistent-random-walk
  right random-normal 0 turn-sd
  roll-right random-normal 0 turn-sd
  forward tcell-step
end

to chemokine-biased-walk
  uphill chemokine
  forward tcell-step
end

; =================== CHEMOKINE SYSTEM ===================
to secrete-chemokines
  ask DCs [
    ask patch-here [
      set chemokine chemokine + 10
    ]
  ]
end

to diffuse-chemokines
  diffuse chemokine 0.25
  ask patches [
    set chemokine chemokine * 0.99  ; gradual decay
  ]
end

; ================== DC-T INTERACTIONS ==================
to identify-DCbound-tcells
  ask DCs [
    create-links-to tcells with [color != blue] in-radius 2 [
      set color red
      set tDCfree 3  ; bound for 3 minutes
      
      let dc-color [color] of myself
      ask end2 [
        ; Track encounter history
        let was1 met1?
        let was2 met2?
        
        if dc-color = red [
          if not met1? [
            set met1? true
            set nMeetDC1 nMeetDC1 + 1
          ]
        ]
        
        if dc-color = orange [
          if not met2? [
            set met2? true
            set nMeetDC2 nMeetDC2 + 1
          ]
        ]
        
        ; Check for cross-reactive encounters
        if (met1? and met2?) and (not (was1 and was2)) [
          set nMeetBoth nMeetBoth + 1
        ]
        
        ; Set binding state
        set color blue
        set label dc-color  ; store DC color for later use
      ]
      
      set nDCmeet nDCmeet + 1
    ]
  ]
end

to update-link-status
  ask links [
    set tDCfree tDCfree - dt
    
    if tDCfree < 0 [
      ask end2 [
        ; Color T cell based on encounter history
        if met1? and met2? [
          set color violet  ; met both DCs
        ] [
          if met1? [ set color red ]     ; met DC1 only
          if met2? [ set color orange ]  ; met DC2 only
        ]
        if not met1? and not met2? [ set color green ]  ; met neither
        
        set label ""  ; clear stored DC color
      ]
      die  ; remove the link
    ]
  ]
end
