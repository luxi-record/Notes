1. 三个点的加载动画

   ```html
   <span class='dotting'></span>
   
   <style>
     .dotting {
       display: inline-block; 
       min-width: 2px; 
       min-height: 2px;
       box-shadow: 2px 0 red, 6px 0 red, 10px 0 red;
       animation: dot 4s infinite step-start both;
     }
     @keyframes dot {
       25% { box-shadow: none; }                                  
       50% { box-shadow: 2px 0 currentColor; }                    
       75% { box-shadow: 2px 0 currentColor, 6px 0 currentColor;}
   	}
   </style>
   ```

   

